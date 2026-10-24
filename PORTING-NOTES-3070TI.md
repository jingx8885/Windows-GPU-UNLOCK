# GA104 / RTX 3070 Ti 移植总结（Windows GPU UNLOCK）

> 中文为主的移植记录；技术名词保留英文。对应代码见分支 `profile-ga104`，上游 PR: https://github.com/impacifist/Windows-GPU-UNLOCK/pull/5

## 1. 这个项目是什么

- **Zagitta/gpu-unlock**（上游，Linux 内核模块）：把 Ampere 消费卡的 FP16/BF16 指令发射率解锁到专业卡水平（约 2x）。GA104 已在 `supported_ids` 中。
- **impacifist/Windows-GPU-UNLOCK**（Windows 移植）：同一个事务放到 UEFI 阶段做——固件启动时执行一次 FWSEC/WPR2/FLR + 选择子写入，然后链式加载 Windows Boot Manager。Windows 驱动看到的已经是解锁后的卡。
- 本仓库（jingx8885 fork）：把原方案从唯一支持的 **ASUS B650M / RTX 3090 FE（AMD 平台）** 移植到 **ASUS TUF B660M-E D4 / RTX 3070 Ti（Intel 无核显平台）**，以编译期 profile 的方式与原方案共存。

## 2. 移植的本质：替换一台机器的完整指纹

这个方案不是通用驱动，是 exact-machine 的一次性事务。每换一台机器要重新采集并固化以下常量（本机值）：

| 项 | 本机（GA104 / B660M） | 原机（GA102 / B650M） |
|---|---|---|
| GPU PCI ID | `10de:2482` | `10de:2204` |
| Subsystem | `1462:5052`（MSI） | NVIDIA FE |
| BDF | `01:00.0` | 不同 |
| BAR0 | `0x82000000` | 不同 |
| Chip ID | `0xb74000a1`（GA104） | `0xb74000a0`（GA102） |
| VRAM 寄存器 | `0x2000`（8 GB） | 24 GB |
| PROM SHA-256 | `f429aa1d…2b4df0` | 不同 |
| 根端口 device path | `Pci(1,0)` | 不同 |
| 板卡 / BIOS 门禁 | `TUF GAMING B660M-E D4` / `4003` | B650M / 指定 BIOS |
| EFI 驱动指纹 | `nvgop-ga1xx` 0x26ae0、AMI EDID runtime 0x9000、AMI combo splitter | 各自 size/stop/version/.text sha256 |

## 3. 关键差异：控制台拓扑（本次移植最大的改动）

原机路径假定有**第二个显示源**（two-child nvgop console）。本机 i7-12700F **没有核显**，拓扑不同：

- `nvgop` 独占 GPU 的 PCI function；
- 控制台画面由 **AMI GOP combo splitter 的 aggregate GOP** 提供；
- 因此新增 `GU_CONSOLE_AGGREGATE` 路径：wrapper 按指纹依次释放 nvgop → EDID runtime → combo splitter → text-out 链，再执行事务。

## 4. 移植流程（可复刻路径）

1. **只读采集**：用 `-DGU_DIAGNOSTIC_PROBE` 构建 `internal-boot/boot/probe.c`，F8 受控启动，抓 PCI config space、capability 链表、BAR 描述符、PROM 哈希、已加载 EFI 镜像 dump。无任何写操作。
2. **固化 profile**：常量集中到 `internal-boot/unlock/machine-profile.h`；`-DGU_PROFILE_B660M_3070TI` 选择本 profile，`-DGU_PROFILE_B650M_3090FE` 保持默认，原机代码路径原样保留。
3. **脚本贯通**：`build.ps1` / `build-unlock.ps1` / `Configure.ps1` / `Headless.ps1` 加 `-HardwareProfile`；安装器的板卡/BIOS 门禁通过 `Get-HeadlessProfileIdentity` 跟随构建 profile。
4. **F8 受控验证**：wrapper 释放 console 链 → 一次自动事务（SS0→`0x88888888`、SS1 `0x7`→`0x8`、PCIe FLR）→ 链式启动 Windows → 显示恢复。
5. **晋升自动冷启**：固件启动顺序改为 headless→Windows Boot Manager，之后每次冷启动自动解锁。
6. **性能验证**：cuBLAS GEMM n=4096、FP32 累加、30 次取中位。

## 5. 验证结果

- 事务日志：`SS0=0x88888888 SS1=0x8 BME=0 maps=0 FLR=1 quiesced=1 fault=0 retained=0 clean=1`
- 二进制可复现：`HEADLESS.EFI` `ce9e3236…` / child `67859b35…`，从 `80a6203` 源码构建逐字节一致
- 性能：**FP16 89.24 / BF16 89.25 / FP32 14.89 TFLOPS**（锁定态同机约 43.7，符合 ~2x 预期）
- 证据与格式要求见 issue #4（compatibility report 模板）、上游 Zagitta/gpu-unlock#32

## 6. 踩过的坑（复刻必读）

- **rebase 到 v0.3.1**：上游把 child 计数改成动态发现（`GU_HEADLESS_MAX_DISPLAY_CHILDREN`），aggregate 分支必须显式 `Full.ChildCount=1`，否则 `repeats=0` 跳过 EDID/GOP 释放，F8 路径在 `full-release-recheck` **安全失败**（不写任何寄存器）——这是真机复验抓出来的真实缺陷。
- **无核显是 aggregate 拓扑的前提**：带核显的 CPU（非 F 后缀）应走另一条路径，别混用。
- **S3 睡眠会丢解锁**：没有 resume 路径，睡眠后需重启重新过 UEFI。
- **前置条件**：Secure Boot 关、C 盘不加密（BitLocker 关）、专用 FAT32 ESP。
- **部署细节**：`Headless.ps1` 的 StageFile 遇到同名不同 hash 会拒覆盖，先改名旧文件；`Remove-Item` 对 `U:`（ESP 挂载点）可能被策略拦截，用 `Move-Item` 改名代替。
- **每次构建都是 exact-machine**：往树里加文件就会改变 PE 字节，所以 B650M profile 在本分支重出不同 hash 是预期行为，不是回归。

## 7. 换别的机器怎么办

- 同板卡同 BIOS 同卡：直接用 `b660m-3070ti` profile 构建部署。
- 任何硬件差异：回到第 4.1 步用 diagnostic probe 重新采集全套指纹，新建 profile。**改常量不等于完成移植**——必须走一遍 F8 受控验证 + 净化日志 + 基准测试，按官方 compatibility report 格式留证据。
- 所有失败路径都是 fail-safe：指纹不匹配、release 链不完整、PROM 对不上都会在写寄存器之前中止。

## 8. 命令速查

```powershell
# 构建（本机 profile）
powershell -ExecutionPolicy Bypass -File research\headless\build-unlock.ps1 `
  -Clang C:\tools\LLVM20\bin\clang.exe -LldLink C:\tools\LLVM20\bin\lld-link.exe `
  -Python C:\tools\py314\python.exe -HardwareProfile b660m-3070ti

# 部署到 ESP
internal-boot\Headless.ps1 -Assets C:\work\assets

# 回退：恢复 Windows 直启
internal-boot\Headless.ps1 -RestoreWindowsFirst
```

## 9. 链接

- PR（官方合并中）: https://github.com/impacifist/Windows-GPU-UNLOCK/pull/5
- 兼容性报告: https://github.com/impacifist/Windows-GPU-UNLOCK/issues/4
- 上游兼容性报告: https://github.com/Zagitta/gpu-unlock/issues/32
