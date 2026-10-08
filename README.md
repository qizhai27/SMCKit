SMCKit
======

用 Swift 编写的 Apple SMC（System Management Controller）库与命令行工具。通过与
AppleSMC.kext（SMC 的私有内核驱动）通信，实现读取温度传感器、读取/设置风扇转速
（RPM）等功能。

本版本在 [beltex/SMCKit](https://github.com/beltex/SMCKit) 基础上做了现代化改造，
**同时支持 Intel（含 T2 机型）与 Apple Silicon Mac**。

- 库的使用示例可参考 [dshb](https://github.com/beltex/dshb)，一个用 Swift 编写的
  macOS 系统监视器
- 其他 macOS 系统统计信息可参考
  [SystemKit](https://github.com/beltex/SystemKit)


### 功能特性

- 读取 CPU / GPU / 机箱等温度传感器（已知映射与未知传感器）
- 读取风扇数量、当前 / 最低 / 最高转速
- 手动设置风扇转速，支持一键全速（`--max`）
- 恢复 macOS 自动风扇控制
- 读取电源（电池 / 适配器）与其他杂项信息
- 查询任意 SMC key 是否存在
- 通用二进制（universal binary，x86_64 + arm64），一个二进制通吃 Intel 与
  Apple Silicon
- 自动适配不同机型的 SMC 数据编码（FPE2 / `flt`）与风扇控制协议


### System Management Controller

_"The System Management Controller (SMC) is an internal subsystem introduced by
Apple Inc. with the introduction of their new Intel processor based machines
in 2006. It takes over the functions of the SMU. The SMC manages thermal and
power conditions to optimize the power and airflow while keeping audible noise
to a minimum. Power consumption and temperature are monitored by the operating
system, which communicates the necessary adjustments back to the SMC. The SMC
makes the changes, slowing down or speeding up fans as necessary."_
-via Wikipedia

更多资料：

- [System Management Controller](https://en.wikipedia.org/wiki/System_Management_Controller)
- [System Management Unit](https://en.wikipedia.org/wiki/System_Management_Unit)
- [Power Management Unit](https://en.wikipedia.org/wiki/Power_Management_Unit)


### 平台兼容性

| 机型 | 温度读取 | 风扇读取 | 风扇设置协议 |
|------|---------|---------|-------------|
| 老款 Intel（无 T2） | ✅ | ✅（`fpe2` 编码） | 写 `F{id}Mn`（最低转速） |
| Intel + T2（如 MacBookPro16,1） | ✅ | ✅（`flt` 小端浮点编码） | 写 `F{id}Md`（模式）+ `F{id}Tg`（目标转速） |
| Apple Silicon（M1–M4） | ✅ | ✅ | `F{id}Md` + `F{id}Tg`；M3/M4 Pro/Max 需先写 `Ftst=1` 解锁 |
| Apple Silicon（M5） | ✅ | ✅ | 小写 `F{id}md` + `F{id}Tg`，无 `Ftst` |

程序会在运行时查询 key 信息（`keyInformation`）自动选择正确的编码与协议，
无需手动指定机型。


### 编译要求

- Xcode（在 Xcode 26 / Swift 6 工具链上验证通过，以 Swift 4 语言模式编译）
- macOS 部署目标：10.13 及以上


### 获取源码与安装

clone 时务必加 `--recursive` 以初始化子模块（CommandLine CLI 库）：

```sh
git clone --recursive https://github.com/beltex/SMCKit
```

如果已经 clone，在项目目录执行：

```sh
git submodule update --init
```

编译并安装到 `/usr/local/bin`（同时安装 man page 到
`/usr/local/share/man/man1`）：

```sh
make install
```

> **签名提示**：构建过程中的 `strip` 会破坏 ad-hoc 代码签名，而 arm64（Apple
> Silicon）强制要求有效签名，否则内核会拒绝运行。若构建后运行报
> "killed"/签名无效，请重新签名：
>
> ```sh
> codesign -s - --force bin/smckit
> ```

卸载：

```sh
make uninstall
```


### 使用说明

#### 总览（不带任何参数）

直接运行 `smckit` 会依次打印温度、风扇、电源与杂项的全部信息：

```sh
$ smckit
-- Temperature --
CPU_0_DIE               70.0°C
CPU_0_PROXIMITY         64.0°C
...
-- Fan --
[id 0] Fan 0
        Min:      1836 RPM
        Max:      5297 RPM
        Current:  1834 RPM
[id 1] Fan 1
        Min:      1700 RPM
        Max:      4905 RPM
        Current:  1708 RPM
-- Power --
AC Present:       true
Battery Powered:  false
Charging:         false
Battery Ok:       true
Battery Count:    1
-- Misc --
Disc in ODD:      false
```

> 部分机型（如带 T2 的 MacBook Pro）不提供风扇名称 key（`F{id}ID`），
> 风扇名会显示为兜底名称 `Fan 0`、`Fan 1`。

#### 选项一览

| 选项 | 说明 |
|------|------|
| `-a`, `--fan-auto` | 将指定风扇（`-n`）恢复为 macOS 自动控制 |
| `-c`, `--color` | 对输出着色（按温度 / 转速告警等级） |
| `-d`, `--display-keys` | 打印温度时同时显示对应的 SMC key（FourCC） |
| `-f`, `--fan` | 显示风扇转速（RPM） |
| `-h`, `--help` | 显示帮助 |
| `-k <KEY>`, `--check-key <KEY>` | 检查某个 FourCC 在本机是否为有效 SMC key |
| `-m`, `--misc` | 显示杂项信息（如光驱中是否有光盘） |
| `-n <ID>`, `--fan-id <ID>` | 指定风扇编号（从 0 开始），设置转速时必填 |
| `-p`, `--power` | 显示电源相关信息 |
| `-s <RPM>`, `--fan-speed <RPM>` | 将指定风扇设置到目标转速（需 root，与 `-x` 互斥） |
| `-x`, `--max` | 将指定风扇直接设为该风扇硬件最大转速（需 root，与 `-s` 互斥） |
| `-t`, `--temperature` | 显示硬件映射已知的温度传感器 |
| `-u`, `--unknown-temperature-sensors` | 显示硬件映射未知的温度传感器 |
| `-v`, `--version` | 显示版本号 |
| `-w`, `--warn` | 显示温度 / 风扇转速的告警等级（Cool/Nominal/Danger/Crisis） |

#### 常用示例

查看风扇状态：

```sh
smckit -f
```

查看温度（带 SMC key 与告警等级、彩色输出）：

```sh
smckit -t -d -w -c
```

把 0 号风扇设置为 5000 RPM（**需要 root**）：

```sh
sudo smckit -n 0 -s 5000
```

让 0 号风扇直接全速运转（自动读取该风扇最大转速，**需要 root**）：

```sh
sudo smckit -n 0 -x
```

恢复 0 号风扇为 macOS 自动控制（**需要 root**）：

```sh
sudo smckit -a -n 0
```

检查某个 SMC key 是否存在：

```sh
$ smckit -k F0Ac
F0Ac is a valid SMC key on this machine
```

设置转速时的典型成功输出（T2 / Apple Silicon 协议）：

```text
Fan target speed set successfully (manual mode)
[id 0] Fan 0
        Min:             1836 RPM
        Max:             5297 RPM
        Target:          5000 RPM
        Current:         1834 RPM

To return to automatic control: smckit -a -n 0
```

#### 参数约束

- `-s` 与 `-x` 互斥，同时使用会报错
- `-s` / `-x` / `-a` 都必须配合 `-n <ID>` 使用
- 目标转速必须 `> 0` 且不超过该风扇最大转速，否则报
  `Invalid fan speed. Must be <= max fan speed`
- 写操作（设置转速 / 全速 / 恢复自动）必须以 root 身份运行，否则报
  `This operation must be invoked as the superuser`


### 复制二进制到其他 Mac

`smckit` 编译产物是 universal binary（含 x86_64 与 arm64 两个架构），
可以直接复制到其他 Intel 或 Apple Silicon Mac 上运行，无需重新编译。

但由于使用的是 **ad-hoc 签名**（无开发者证书），跨机器复制后建议重新签名：

```sh
codesign -s - --force /path/to/smckit
```

如果二进制带有 quarantine 属性（经浏览器 / AirDrop 传输），首次运行可能被
Gatekeeper 拦截，可在"系统设置 → 隐私与安全性"中放行，或执行：

```sh
xattr -d com.apple.quarantine /path/to/smckit
```


### 注意事项

- **操作硬件有风险**：设置风扇转速会直接影响散热，请确保目标转速合理，
  长时间高温下手动限速可能导致硬件降频或损坏。
- 在 T2 / Apple Silicon 机型上，macOS 的 `thermalmonitord` 可能会定期夺回
  风扇控制权，手动模式不一定长期保持，这是系统正常行为；需要时可用
  `smckit -a -n <ID>` 显式恢复自动控制。
- 本质上使用的是私有 API，该库不可能通过 Mac App Store 审核。


### 错误码排查

输出形如 `unknown(kIOReturn: 0, SMCResult: <code>)` 时，常见 SMCResult 含义：

| SMCResult | 含义 | 常见原因与处理 |
|-----------|------|---------------|
| 0 | 成功 | — |
| 132 (0x84) | key 不存在 | 该 SMC key 在当前机型上无效 |
| 134 (0x86) | key 不可写 | key 只读、未解锁，或走错了控制协议（如在 T2 上写 `F{id}Mn`）；需要 root 时也可能出现 |
| 135 (0x87) | 数据尺寸 / 类型不匹配 | 读取时声明的 size/type 与 key 实际类型不符（本工具已自动处理 `fpe2`/`flt` 差异） |


### Library Usage Notes

- The use of this library  will almost certainly not be allowed in the
  Mac App Store as it is essentially using a private API
- If you are creating a macOS command line tool, you cannot use SMCKit as a
  library as Swift does not currently support static libraries. In such a
  case, the `SMC.swift` file must simply be included in your project as another
  source file. See
  [SwiftInFlux/Runtime Dynamic Libraries](https://github.com/ksm/SwiftInFlux#runtime-dynamic-libraries)
  for more information and both SMCKitTool &
  [dshb](https://github.com/beltex/dshb) as examples of such a case.

核心代码位于 [SMCKit/SMC.swift](SMCKit/SMC.swift)，CLI 入口位于
[SMCKitTool/main.swift](SMCKitTool/main.swift)。


### References

There are many projects that interface with the SMC for one purpose or another.
Credit is most certainly due to them for the reference. Such projects as:

- iStat Pro
- [osx-cpu-temp](https://github.com/lavoiesl/osx-cpu-temp)
- [PowerManagement](http://www.opensource.apple.com/source/PowerManagement/)
- [powermetrics(1)](https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man1/powermetrics.1.html)
- [smcFanControl](https://github.com/hholtmann/smcFanControl)

Handy I/O Kit references:

- [iOS Hacker's Handbook](http://ca.wiley.com/WileyCDA/WileyTitle/productCd-1118204123.html)
- [Mac OS X and iOS Internals: To the Apple's Core](http://www.wiley.com/WileyCDA/WileyTitle/productCd-1118057651.html)
- [OS X and iOS Kernel Programming](http://www.apress.com/apple-mac/objective-c/9781430235361)


### License

This project is under the **MIT License**.


### Fun

While the SMC driver is closed source, the call structure and definition of
certain structs needed to interact with it (see `SMCParamStruct`) happened to
appear in the open source Apple **PowerManagement** project at around version
211, and soon after disappeared. They can be seen in the
[PrivateLib.c](http://www.opensource.apple.com/source/PowerManagement/PowerManagement-211/pmconfigd/PrivateLib.c)
file under `pmconfigd`. In the very same source file, the following snippet can be
found:

```c
// And simply AppleSMC with kCFBooleanTrue to let them know time is changed.
// We don't pass any information down.
IORegistryEntrySetCFProperty( _smc,
                    CFSTR("TheTimesAreAChangin"),
                    kCFBooleanTrue);
```

Almost certainly a reference to Bob Dylan's
<a href="https://en.wikipedia.org/wiki/The_Times_They_Are_a-Changin%27_(song)">The Times They Are a-Changin'</a> :)
