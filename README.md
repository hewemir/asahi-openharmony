# AsahiOpenHarmony

在 Apple Silicon 上自建并运行 OpenHarmony 的板级/vendor 源码与构建方式。

目前**只在 M1 Pro 的 MacBook Pro（MacBookPro18,3）上验证过**。其它 Apple Silicon 机型没有测试，不保证能用。

## 进度

实验性。现在能**直接进入桌面**（没有锁屏这一层），控制中心、通知中心都能正常用。但系统里没有预置应用——设置之类的基础 App 一概没有，字体等资源也还缺。别当成能日常用的系统。

## 关键难点：16KB 页

M1 的硬件页大小是 **16KB**。AGX GPU 的 UAT 页表按 16KB 组织，DART（IOMMU）也只支持 16KB，所以内核必须走 16K 页（这里用的是 asahi 6.8 内核，`CONFIG_ARM64_16K_PAGES=y`，VA_BITS=48）。

麻烦出在用户态：上游 OpenHarmony 的 ELF 是按 **4KB** 对齐编译的。放到 16KB 页的内核上，这种库一 `dlopen` 就会崩在动态链接器里——每个 LOAD 段的文件偏移和虚拟地址不满足 16KB 同余，`mmap` 直接错位（表现为 `ld-musl` 里 `paddr=0` 的 SIGSEGV）。所以这条路的第一件事就是把 ELF 的段对齐改成 16KB：

- 自己能编译的，链接时直接加 `-Wl,-z,max-page-size=16384`，产出的 LOAD 段就是 0x4000 对齐。
- 只有二进制的，做重排（realign）：把 `p_offset` 重排到与 `p_vaddr` 在 16KB 下同余，`p_align` 改成 0x4000，不动虚拟地址和重定位。

## 其它要攻的点

**仅供参考**

- **内核**：asahi 6.8.8，带 Apple DRM/AGX 驱动，16K 页。
- **显示/GPU**：Apple 侧没有现成 HD/HDI，需要自己补 vendor 侧的 display composer、gfx 桩，并把 DCP 拉起来。
- **构建工具链**：自建 OpenHarmony 5.0.3（产品名 `macbookpro18_3`），用 OHOS clang 15 / GN / Ninja。系统里的 Python 版本较新，`build_utils.py` 里的 `pipes` 之类的旧模块要去掉、`ninja` 的重复 target 要用 `-w dupbuild=warn` 绕过。
- **页对齐贯穿全程**：kernel、userspace、甚至自己写的桩库，只要忘了 16K 对齐就会在加载阶段挂。

## 目录

- `device/board/apple/macbookpro18_3/` — 板级配置：kernel 配置、init cfg、fstab、bootanimation 等
- `device/soc/apple/t600x/` — SoC 侧，display HAL 等
- `vendor/apple/macbookpro18_3/` — vendor 配置：hdf_config、window_config、image_conf 等
- `patches/ohos-5.0.3/` — 对上游 OpenHarmony 5.0.3 的补丁

## 构建

基线：OpenHarmony 5.0.3 源码 + 上面的板级/vendor 源码，配 16K 页的 asahi 6.8 内核，用 OHOS clang 15 / GN / Ninja 编译。

## 许可

本仓库自有部分用 Apache License 2.0，见 `LICENSE`。部分文件来自 OpenHarmony，保留其原始版权头。另见 `NOTICE`。

---

感谢 DeepSeek、Gemini 和 Claude 在开发过程中的帮助。
