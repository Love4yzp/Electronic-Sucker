# AGENTS.md

给在本仓库工作的 agent 的约定。项目全貌、设计决策与缺陷记录见 [Project.md](Project.md)，
安装与使用见 [README.md](README.md)。本文只写「怎么改代码才不会出事」。

## 项目一句话

浏览器网页 → Python 桥接（`bridge/bridge.py`）→ 串口 115200 文本协议 →
Arduino Uno 固件（`firmware/sucker/sucker.ino`）→ 舵机式 PWM 驱动气泵与电磁阀。
控制的是真实物理设备，气泵空转会烧，安全兜底（看门狗）在固件里，不在网页。

## 验证命令

| 场景 | 命令 |
|---|---|
| 编译固件（不上传） | `arduino-cli --config-file arduino-cli.yaml compile --fqbn arduino:avr:uno firmware/sucker` |
| 编译并烧录 | `./scripts/flash.sh [串口]` |
| 固件层测试（需接板子） | `python3 tools/serial_test.py --seq` |
| 看门狗回归压测 | `python3 tools/serial_test.py --stress 40`，`"ev":"watchdog"` 次数必须为 0 |
| 端到端（需服务已运行） | `python3 tools/ui_test.py` |

**任何行为改动后至少跑一遍 `--stress 40` + `ui_test.py`**（Project.md §12.3）。
没有硬件时至少保证固件编译通过、Python 无语法错误。

## 规则表（每条规则必须有强制手段，只靠文档的规则等于没有）

| 规则 | 强制手段 |
|---|---|
| 改行为必须同步 Project.md | review 检查；Project.md 页脚声明 |
| 看门狗超时 ≥ 6000 ms（心跳 1500 ms 的 4 倍） | 桥接 `clamp_watchdog()`；**固件侧目前无下限，改 `SET watchdog` 处理时保持或补上** |
| 改引脚只改固件顶部 `PIN_PUMP` / `PIN_VALVE` 常量 | 改完后必须同步 README 接线表、Project.md §2.1、`web/index.html` 接线说明（共 4 处，目前是手写副本，以固件常量为准） |
| 修 bug 先写失败的回归测试再改实现 | 使用 `.agents/skills/tdd`；参照 Project.md §9.1 的修法 |
| 测试断言用唯一标识（如 `"ev":"watchdog"` 而非 `watchdog`） | Project.md §9.4 的教训；review 检查 |
| 输出信号默认舵机式 PWM，不要改成 `digitalWrite` | Project.md §2.2；厂家模块收的是舵机信号 |
| 服务只绑定 `127.0.0.1` | review 检查；`--host 0.0.0.0` 仅限用户显式要求 |

## 固件改动的两个雷区（来自实测缺陷，详见 Project.md §9）

1. **`loop()` 里 `millis()` 的取值必须在串口处理之后**，且看门狗比较用有符号强转
   `(long)(now - lastCmdAt)`。挪动 `now` 的位置会让看门狗在刚收到命令时随机误触发。
2. 心跳间隔和看门狗超时是一对，改任意一个都要重新核算余量。

## 项目内可用的 skill（`.agents/skills/`，选自 pstack，许可见 PSTACK-LICENSE）

三个，各管一个时刻：

| Skill | 什么时候用 |
|---|---|
| `tdd` | 修 bug 时：先写失败的重现测试，再修 |
| `correct` | 同一类错误第二次出现时：在最高层级（架构 > 类型 > lint/CI > 测试 > 文档）消灭它，并更新上面的规则表 |
| `interrogate` | 交付前：对 diff 做多模型对抗评审（本仓库控制物理设备，值得） |

上面的「验证命令」表同时就是必须被证明的用户可见行为清单，不另建功能地图。
