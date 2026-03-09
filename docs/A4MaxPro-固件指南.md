# Anycubic 4Max Pro — Marlin 2.0.x 固件指南

> 适用固件：[Marlin-A4MaxPro-2.0.x](https://github.com/HackySoftOfficial/Marlin-A4MaxPro-2.0.x)  
> 当前配置：X/Y/Z 轴 TMC2208_STANDALONE 静音驱动，无 BLTouch

---

## 目录

1. [固件特性](#固件特性)
2. [编译固件](#编译固件)
3. [刷写固件](#刷写固件)
4. [刷写后必做操作](#刷写后必做操作)
5. [校准步骤](#校准步骤)
6. [Cura 配置](#cura-配置)
7. [起始 G-code](#起始-g-code)
8. [结束 G-code](#结束-g-code)
9. [常见问题](#常见问题)

---

## 固件特性

| 功能 | 说明 |
|---|---|
| **Marlin 2.0.x** | 从原厂 RC8 升级，稳定性与功能大幅提升 |
| **修复热敏电阻** | 原厂热端温度传感器配置有误，已修正 |
| **TMC2208 静音驱动** | X/Y/Z 三轴静音，打印噪音大幅降低 |
| **网格调平 (MBL)** | 25 点手动网格调平，补偿不平整热床 |
| **Linear Advance** | 改善拐角溢料/欠料，提升打印质量 |
| **断料检测** | 耗材耗尽自动暂停，换料后继续打印 |
| **智能暂停** | 暂停时打印头移到安全位置，维持温度 |
| **PID 温控** | 热端与热床均采用 PID 精确温控 |

---

## 编译固件

### 环境要求

- [Visual Studio Code](https://code.visualstudio.com/)
- [PlatformIO 插件](https://platformio.org/install/ide?install=vscode)

### 编译命令

```powershell
$pioexe = "E:\GitHub\Marlin-A4MaxPro-2.0.x\.venv\Scripts\pio.exe"
Set-Location E:\GitHub\Marlin-A4MaxPro-2.0.x
& $pioexe run -e mega2560
```

### 编译成功输出示例

```
RAM:   [=======   ]  74.8% (used 6130 bytes from 8192 bytes)
Flash: [=====     ]  52.4% (used 133052 bytes from 253952 bytes)
========================= [SUCCESS] =========================
```

### 固件输出路径

```
E:\GitHub\Marlin-A4MaxPro-2.0.x\.pio\build\mega2560\firmware.hex
```

### 主要配置项（Configuration.h）

| 参数 | 当前值 | 说明 |
|---|---|---|
| `BAUDRATE` | 115200 | 串口波特率 |
| `X_BED_SIZE` | 270 mm | 打印床 X 尺寸 |
| `Y_BED_SIZE` | 205 mm | 打印床 Y 尺寸 |
| `Z_MAX_POS` | 205 mm | Z 轴最大高度 |
| `DEFAULT_AXIS_STEPS_PER_UNIT` | X:100 Y:80 Z:800 E:418 | 各轴步进数/mm |
| `X/Y/Z_DRIVER_TYPE` | TMC2208_STANDALONE | 步进驱动型号 |
| `DEFAULT_Kp/Ki/Kd` | 18.53 / 1.27 / 67.55 | 热端默认 PID 值 |

---

## 刷写固件

### 使用 Ultimaker Cura 刷写

1. 安装 [CP210x USB 驱动](https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers)（若电脑无法识别串口）
2. USB-B 线连接打印机与电脑，打开打印机电源
3. 打开 Cura → **设置 → 打印机 → 管理打印机**
4. 选中打印机 → 点击 **升级固件**
5. 选择 **从磁盘上传自定义固件**
6. 选择文件：`.pio\build\mega2560\firmware.hex`
7. 等待进度条完成（约 30~60 秒），打印机自动重启

> 如果找不到升级固件选项，在 Cura Marketplace 中安装 **Firmware Updater** 插件。

### 使用 avrdude 命令行刷写

```powershell
# 将 COM3 替换为实际端口号
avrdude -p m2560 -c wiring -P COM3 -b 115200 -D -U flash:w:firmware.hex:i
```

---

## 刷写后必做操作

通过 Cura 监控面板或 Pronterface 以 **115200 波特率**连接，依次发送：

```gcode
M502    ; 重置所有参数为固件默认值（必须执行，清除旧 EEPROM 数据）
M500    ; 保存到 EEPROM
```

**完成后断电重启**（不能只点软件重启）。

---

## 校准步骤

### 1. 热端 PID 自动校准（必做）

```gcode
M106 S204           ; 开启冷却风扇，模拟实际打印状态
M303 E0 S205 C8 U1  ; PLA 205°C 自动校准，8 轮，完成后自动应用
M500                ; 保存结果
```

热床 PID（如热床温度不稳定时执行）：

```gcode
M303 E-1 S60 C6 U1  ; 热床 60°C 校准
M500
```

### 2. 挤出机步进校准（必做）

当前固件默认 E 步数：**418 步/mm**

校准方法：

1. 在挤出机进料管入口处标记 **120mm** 处
2. 发送命令挤出 100mm：
   ```gcode
   M83          ; 相对模式
   G1 E100 F100 ; 挤出 100mm
   ```
3. 量剩余长度，应为 20mm。若不符，计算新步数：

$$E_{新} = E_{旧} \times \frac{100}{100 - (实际剩余 - 20)}$$

4. 应用并保存：
   ```gcode
   M92 E<新数值>
   M500
   ```

### 3. 网格床面调平（推荐）

1. 打印机屏幕：**Extra Menu → Start Mesh Leveling**
2. 共 25 个点，每点用 A4 纸张法调整高度（纸张能感受轻微阻力即可）
3. 全部完成后选择 **Save EEPROM**

验证调平是否已保存：
```gcode
M420 V    ; 查看当前网格数据
```

### 4. Linear Advance K 值校准（推荐）

使用 [K-factor 校准工具](https://marlinfw.org/tools/lin_advance/k-factor.html) 生成测试 G-code。

**工具参数填写：**

| 参数 | 值 |
|---|---|
| Printer type | Cartesian |
| Bed size X/Y | 270 / 205 |
| Nozzle diameter | 0.4 mm |
| Filament diameter | 1.75 mm |
| Start K / End K / Step | 0 / 1.0 / 0.05 |

打印测试图后，找到直线段**宽度最均匀、拐角无溢料无缩料**的行，读取对应 K 值。

应用并保存：
```gcode
M900 K0.2   ; 替换为实际校准值
M500
```

**各材料参考 K 值范围：**

| 耗材 | K 值范围 |
|---|---|
| PLA | 0.1 ~ 0.3 |
| PETG | 0.2 ~ 0.5 |
| ABS | 0.1 ~ 0.3 |
| TPU | 0（建议关闭）|

### 5. Z 轴偏移微调

```gcode
G28          ; 归零
M851 Z0      ; 清零偏移
G1 Z0 F300   ; 移到 Z=0
; 纸张测试，调整偏移值
M851 Z-0.1   ; 示例值，按实际调整
M500
```

---

## Cura 配置

### 打印机参数（管理打印机 → 机器设置）

| 项目 | 值 |
|---|---|
| X 宽度 | 270 mm |
| Y 深度 | 205 mm |
| Z 高度 | 205 mm |
| 加热床 | ✓ 启用 |
| G-code 风格 | RepRap |

### 切片参数建议

| 参数 | 建议值 | 说明 |
|---|---|---|
| 外墙打印速度 | 40~50 mm/s | 过慢会导致积热溢料 |
| 打印加速度 | 500~700 mm/s² | TMC2208 可承受，比原版提升 |
| Z 缝对齐 | Sharpest Corner | 缝隙藏在模型不显眼处 |
| Outer Wall Wipe Distance | 0.2 mm | 消耗外墙末端余料 |
| Enable Coasting | **关闭** ✗ | 与 Linear Advance 冲突 |
| Enable Combing | Not in Skin | 减少表面拉丝 |

### 预热温度

| 材料 | 热端 | 热床 |
|---|---|---|
| PLA | 190°C | 60°C |
| PETG/ABS | 240°C | 80°C |

---

## 起始 G-code

```gcode
G21                          ; 公制单位
G90                          ; 绝对坐标模式
M82                          ; 挤出机绝对模式
M107                         ; 关闭风扇

; ── 归零与调平 ──────────────────────────────
G28                          ; 全轴回零（X Y Z）
M420 S1                      ; 启用网格调平补偿
M900 K0.2                    ; Linear Advance K值（按你校准的结果修改）

; ── 擦嘴 ────────────────────────────────────
G1 Z5 F3000                  ; 抬升Z轴防止刮床
G1 X-3 Y40 F6000             ; 移到刷子位置
G1 X-3 Y5  F3000             ; 擦嘴
G1 X-3 Y40 F3000             ; 擦嘴
G1 X-3 Y5  F3000             ; 擦嘴

; ── 打印前引线 ──────────────────────────────
G1 Z0.3 F1200                ; 降到打印高度
G1 X0   Y5  F3000            ; 移到床边起点
G92 E0                       ; 重置挤出量
G1 X0   Y60 E6   F800        ; 打印引线第一段
G1 X0.5 Y60      F3000       ; 轻微偏移
G1 X0.5 Y5  E12  F800        ; 打印引线第二段（双线更可靠）
G92 E0                       ; 再次重置挤出量

; ── 开始打印 ────────────────────────────────
G1 Z1.0 F3000                ; 抬升Z轴准备移动
M117 Printing...
```

---

## 结束 G-code

```gcode
; ══ 立刻离开模型（最高优先级）════════════════
M400                         ; 等待当前动作完成
G92 E0
G1 E-3 F1800                 ; 快速回抽 3mm，立刻释放喷嘴压力
G91                          ; 相对坐标
G1 Z15 F3000                 ; 抬起 Z 轴 15mm（离开模型顶面）
G90                          ; 切回绝对坐标
G1 X10 Y200 F9000            ; 以最快速度将喷嘴移到床尾远端
M107                         ; 关闭冷却风扇

; ══ 关闭加热，喷嘴已不在模型上方 ═════════════
M104 S0                      ; 关闭热端
M140 S0                      ; 关闭热床

; ══ 等待降温后深度回抽（防漏料） ═════════════
M109 R150                    ; 等待降至 150°C（此时才无流动风险）
G1 E-10 F300                 ; 深度回抽 10mm，彻底防漏

; ══ 结束 ════════════════════════════════════
M84                          ; 关闭步进电机
M300 S2000 P200              ; 提示音
M300 S0    P100
M300 S2000 P200
M117 Print Complete!
```

---

## 常见问题

### 顶层出现溢料凸起
结束 G-code 中 `M400` + `G1 E-3` + `G1 Z15` + `G1 X10 Y200` 这四行必须**在关闭加热之前**执行，确保喷嘴在热端还热时立刻离开模型。

### Z 缝位置不美观
Cura 中将 `Z Seam Alignment` 改为 `Sharpest Corner`，将 Z 缝藏在模型尖角处。

### 更换耗材后重新校准
每次更换不同品牌或材质的耗材，需要重新执行：
- 挤出机步进校准（如果流量有变化）
- Linear Advance K 值校准（不同材质差异较大）
- PID 校准（如果打印温度差距大于 30°C）

### 回退原厂固件
原厂固件（v1.1.7）备份地址：  
`https://drive.google.com/file/d/1FwKHQcOxPabLgirkihu3LnBMuHuZLqZR/view`  
刷入方式同上，替换 hex 文件即可。
