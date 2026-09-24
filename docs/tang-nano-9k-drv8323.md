# Tang Nano 9K + DRV8323 移植说明

原 README 描述的是 Altera Cyclone IV、MP6540 和 AD7928 那套示例。本文是把它迁到 Sipeed Tang Nano 9K，并改接 DRV8323 驱动板和 AS5600 的说明。电流环算法保持 `RTL/foc/` 不动。

DRV8323 手册依据：TI `DRV832x` 数据手册（2026 年文本，产品页 `DRV8323`）。

## 边界

`foc_top` 对外仍然只认这些信号：

| 信号 | 方向 | 含义 |
|---|---|---|
| `phi[11:0]` | 输入 | AS5600 机械角，0–4095 |
| `sn_adc` | 输出 | 一个时钟的脉冲，表示可以开始采三相电流 |
| `en_adc` | 输入 | 一个时钟的脉冲，表示三路 12 bit 同时有效 |
| `adc_a/b/c[11:0]` | 输入 | A/B/C 相电流的 ADC 码 |
| `pwm_a/b/c` | 输出 | 1 = 上管通，0 = 下管通 |
| `pwm_en` | 输出 | 0 = 六管关断 |
| `id/iq`、`id_aim/iq_aim` | — | 电流环监测与目标 |

换开发板只改时钟和管脚。换驱动板只改 `foc_top` 外面的 PWM 与电流适配。Clark、Park、PI、SVPWM 不改。

## Tang Nano 9K

芯片是高云 GW1NR-LV9QN88PC6/I5：8640 个 LUT4、6480 个触发器、26 块共 468 Kbit 块存储器、20 个 18×18 乘法器、2 个 PLL。板载晶振 27 MHz，接 FPGA 第 52 脚。下载和串口由板载 BL702 经 USB 完成。

主时钟目标 **36 MHz**，用 Gowin `rPLL` 的 IP 向导生成，`locked` 作为 `rstn`。SVPWM 频率是 `clk/2048`，约 17.6 kHz。VCO 必须落在芯片允许范围内，不能把 `27×4/3` 直接填成分频参数。

沿用 `adc_ad7928.v` 时，主时钟保持在 40 MHz 以下：该模块把时钟二分频作为 SPI，AD7928 的 SCLK 上限是 20 MHz。换成别的 ADC 后，按那颗芯片的 SCLK 上限重算。

36 MHz 下需要改的分频：

- `uart_monitor` 的 `CLK_DIV = 312`（约 115385 bit/s，误差约 0.16%）
- I2C 的 `CLK_DIV` 可维持 10，SCL 约 900 kHz，低于 AS5600 的 1 MHz 上限

`uart_tx` 接到第 17 脚，走板载 USB 串口，格式仍是 115200 8N1。

下列脚是建议，焊接前对照板子丝印。避开 Bank3 的第 79–86 脚（1.8 V），也避开 HDMI 的第 68–77 脚（带上拉）。

| 信号 | 建议脚 | 方向 |
|---|---|---|
| `i2c_scl` / `i2c_sda` | 55 / 54 | SCL 输出，SDA 三态 |
| `INHA` / `INHB` / `INHC` | 29 / 30 / 33 | 输出，接 `pwm_a/b/c` |
| `INLA` / `INLB` / `INLC` | 35 / 40 / 41 | 输出，见下方 6 路译码 |
| `DRV_EN` | 42 | 输出，高有效 |
| `FAULT` | 51 | 输入 |
| `spi_ss` / `spi_sck` / `spi_mosi` / `spi_miso` | 25 / 26 / 27 / 28 | 电流 ADC |
| `uart_tx` | 17 | 板载 USB-UART |

排针是未焊接的 2.54 mm 焊盘。电机电源走驱动板自己的供电，不从板子 USB 的 5 V 取。

综合时确认三张 `case` 查找表进了块存储器。`cartesian2polar.v` 是 4096×18 bit，若被摊成 LUT，8640 个 LUT 会变紧。乘法器会接近 20 个 18×18 的上限，多出来的会落到 LUT 上。这两项以综合报告为准，下面没有实测占用。

这一层要改的文件：

| 文件 | 动作 |
|---|---|
| `RTL/fpga_top.v` | 去掉 Altera `altpll`，改为高云 `rPLL`；按上表加管脚；电流码按下一节取反 |
| 新建 `tang_nano_9k.cst` | 管脚和电平 |
| `uart_monitor` 的 `CLK_DIV` | 改为 312 |
| `RTL/foc/*` | 不改 |
| `i2c_register_read.v` | 不改，只保留现有参数 |

## 驱动板 POWER_DRV8323HR_V1.01

依据产品说明书 Rev.1.00b2（2026-07-13），芯片是硬件接口的 **DRV8323HR**。母线 12–40 V，逻辑接口 3.3 V，峰值电流标注 25 A。P2 是 16×2 排针。`+5.0V` 和 `VREF_3.3V` 都是板子输出，不要从 FPGA 往这两脚灌电。FPGA、驱动板和母线负极必须共地。

出厂 PWM 模式是 **6 路**，不是 3 路：`R49 = 0Ω`、`R45` 不贴，`MODE` 被接到 AGND。说明书同时写明改成 3 路模式后，PWM 只接 `INHx`。六路输入都引到了 P2。

| P2 | 信号 | 方向 | 说明 |
|---|---|---|---|
| 1 | +5.0V | 板子输出 | 不要外灌 |
| 4 / 6 / 8 | IOUTC / IOUTB / IOUTA | 板子 → ADC | C/B/A 相电流，模拟电压 |
| 10 | VREF_3.3V | 板子输出 | REF3033，3.3 V 参考 |
| 11 / 12 / 13 | VBUS / CBUS / NTC | 板子 → ADC | 母线电压、母线电流、温度，FOC 电流环可以先不接 |
| 14 | FAULT | 板子 → FPGA | 低有效，板上已有 4.7 kΩ 上拉到 3.3 V |
| 23 | DRV_EN | FPGA → 板子 | 使能，板上 10 kΩ 下拉，默认低 |
| 24 | CAL | FPGA → 板子 | 电流放大器校准，正常运行保持低 |
| 25 / 26 | INHA / INLA | FPGA → 板子 | A 相上下管 |
| 27 / 28 | INHB / INLB | FPGA → 板子 | B 相上下管 |
| 29 / 30 | INHC / INLC | FPGA → 板子 | C 相上下管 |

保持出厂 6 路模式，不改 `R49`。不要改 `svpwm.v`。在 `fpga_top` 里把每相一根 PWM 再译出三根反相：

| `pwm_en` | `pwm_x` | `INHx` | `INLx` |
|---|---|---|---|
| 0 | X | 0 | 0 |
| 1 | 1 | 1 | 0 |
| 1 | 0 | 0 | 1 |

`foc_top` 复位后 `pwm_a/b/c` 默认为 1。在 6 路模式下，如果这时 `DRV_EN` 已经为高，上管会导通。所以母线接通前六路 PWM、`DRV_EN`、`CAL` 都必须为低；FPGA 把六路驱动到 0 之后，再把 `DRV_EN` 拉高，至少等 1 ms，确认 `FAULT` 为高，然后才允许 `foc_top` 出 PWM。

死区在芯片内部。这颗是硬件版，没有 SPI 死区寄存器。栅极驱动电流出厂为高阻档，源出 120 mA、灌入 240 mA（`R50 = 1 MΩ`，`R46` 不贴）。VDS 保护阈值出厂 1.88 V（`R44 = 18 kΩ`，`R48` 不贴）。

`FAULT` 为低时拉低 `foc_top` 的 `rstn`，否则 PI 会在功率级已经关掉之后继续积分。`DRV_EN` 从低拉高的唤醒过程中，`FAULT` 可能短暂变低，要等唤醒完成后再根据它复位。拉低 `rstn` 会重新做一次约 0.45 s 的吸零。`CAL` 固定输出 0。

逻辑输入与 3.3 V GPIO 兼容。说明书建议 PWM 频率 10–40 kHz，17.6 kHz 落在这个区间里。

## 电流

相电流由 R36/R37/R38（各 4 mΩ）经 DRV8323H 内部放大器送到 P2 的 `IOUTA`/`IOUTB`/`IOUTC`。出厂增益 5 V/V（`R47 = 0Ω`，`R43` 不贴），参考为 `VREF_3.3V`。零电流约在 1.65 V，灵敏度约 `0.02 V/A`（`4 mΩ × 5 V/V`）。25 A 时相对零点偏移约 0.5 V，落在 3.3 V ADC 的线性范围内。

双向公式仍是：

```
I = (VREF/2 − VIOUTx) / (GCSA × RSENSE)
```

驱动板上已经有采样电阻和放大器，不用再做一套电流采样电路。Tang Nano 9K 没有 ADC，FPGA 引脚也不能接 `IOUTx`。在 P2 和 FPGA 之间加一颗 ADC，只做数字化。`IOUTA→adc_a`，`IOUTB→adc_b`，`IOUTC→adc_c`。数字接口仍是 `sn_adc` 开始、`en_adc` 同时提交。母线电压、母线电流和 NTC 不参与这套电流环。

模拟线用短杜邦线即可。接触电阻可以忽略，线太长或贴着电机相线才会耦合进干扰。灵敏度约 0.02 V/A，20 mV 干扰折合 1 A。三路共模会被减法消掉，单相串入的干扰会留在 `iq` 里。`IOUTA/B/C` 各配一根地，长度几厘米，ADC 放在 P2 旁边。PWM、SPI、`DRV_EN`、`FAULT` 用普通杜邦线。

优先用 **AD7928**。`adc_ad7928.v` 和 `fpga_top` 里的通道号已经按它写好：VIN1/2/3 接 `IOUTA/B/C`，其余 VIN 接 AGND。AVDD 用 3.3 V，AGND 接 P2 的地。REF IN 就近接 0.1 µF 到地，用片内 2.5 V 基准，不要外接 `VREF_3.3V`。25 A 时 `IOUT` 约在 1.15–2.15 V，落在 0–2.5 V 量程内。裸片是 TSSOP-20，焊到 2.54 mm 转接板上再接杜邦线，电源脚和 REF IN 的 0.1 µF 放在转接板上。

AD7606 也能用，但要新写驱动，接线更多，电流环也不会更准。模块的 `+5V` 只供电，`VIO`（VDRIVE）接 FPGA 的 3.3 V。`RANGE` 和 `OS0/OS1/OS2` 都接低，用 ±5 V、关闭过采样。

只有下管导通时，`IOUTx` 才代表相电流。`hold_detect` 在三相 PWM 都为低之后延迟 `SAMPLE_DELAY` 再采样，这条时序继续用。`MAX_AMP` 先保持 384。`SAMPLE_DELAY=120` 在 36 MHz 下约 3.3 µs。

`foc_top` 的正方向是电流从半桥流入电机，公式按反相放大器书写。DRV8323 公式里的正电流是从 `SPx` 到 `SNx`（流进半桥）。在 `fpga_top` 里对三路码做 `~adc` 再送进 `foc_top`。用很小的 `iq_aim` 看 `iq` 是否同号跟随；方向反了就去掉取反。`CAL` 保持低，第一版不做放大器校准。

## AS5600

`i2c_register_read` 已经按从机地址 `7'h36`、寄存器 `8'h0E` 读取，丢掉高 4 位得到 12 bit 机械角。模块不用改。编码器模块上要有 I2C 上拉。先把 `phi` 打到串口，转一圈应看到 0–4095 单调变化，再定 `ANGLE_INV` 和 `POLE_PAIR`。

## 上电顺序

按说明书：母线接通前，六路 PWM、`DRV_EN`、`CAL` 都为低。接通 12–40 V 母线后，再把 `DRV_EN` 拉高，至少等 1 ms，确认 `FAULT` 为高，然后才输出 PWM。首次用限流电源。

1. 不接母线。PLL 锁定，串口 115200 8N1 有输出。六路 PWM 保持低，`DRV_EN` 和 `CAL` 为低。
2. 接 AS5600，确认 `phi`。
3. 接 P2 和电机相线，接通母线。`DRV_EN` 置高并等待 1 ms。`FAULT` 为高之后才放开 PWM。吸零时应能把转子吸住。
4. 把 `IOUTA/B/C` 的 12 bit 接进 `en_adc`。`id_aim=0`，`iq_aim` 用很小的值，确认同号跟随后再用示例里的 ±200。
5. 最后调 `Kp`、`Ki`、`ANGLE_INV`、`POLE_PAIR`。

第 4 步之前不要改 `RTL/foc/`。停机时先停 PWM，再拉低 `DRV_EN`，断开母线之后才能拔线。
