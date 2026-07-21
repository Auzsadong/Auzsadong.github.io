---
title: "STM32开发备忘录：使用 HRTIM 定时器生成 PWM 并联动触发 ADC"
date: 2026-07-22 16:30:00
math: true
categories:
  - [嵌入式开发, STM32]
tags: [DAC, HRTIM,TIM, DMA, 笔记, 嵌入式开发]
---

# 使用 HRTIM 定时器生成 PWM 并联动触发 ADC

在数字电源、Boost/PFC、电机控制等功率电子场景里，经常需要在 PWM 的固定相位进行电压或电流采样。如果使用普通定时器中断再由软件启动 ADC，采样点会受到中断响应和代码执行时间影响。更稳妥的做法是让 HRTIM 同时产生 PWM 边沿和 ADC 触发事件：

```text
HRTIM1 Timer A CMP1  -> 控制 PWM 下降沿
HRTIM1 Timer A CMP4  -> 产生 HRTIM ADC Trigger 1
ADC1 Regular Group   -> 由 HRTIM_TRG1 上升沿启动转换
```

本工程基于 STM32G474VETx，使用 STM32CubeMX 6.16.1 生成 CMake 工程。当前配置文件为 `G474_HETIM_ADC.ioc`，主要源码位于 `Core/Src`。
## 一、工程架构

### 1. 目录结构

```text
G474_HETIM_ADC
├── G474_HETIM_ADC.ioc             CubeMX 配置文件
├── CMakeLists.txt                 顶层 CMake 工程
├── CMakePresets.json              CMake 预设
├── cmake/stm32cubemx/CMakeLists.txt
├── Core
│   ├── Inc                        应用头文件
│   └── Src
│       ├── main.c                 应用主逻辑，启动 PWM/ADC，更新占空比
│       ├── hrtim.c                HRTIM1 初始化
│       ├── adc.c                  ADC1 初始化
│       ├── gpio.c                 GPIO 初始化
│       └── usart.c                串口调试输出
├── Drivers                        HAL 与 CMSIS 驱动
├── startup_stm32g474xx.s          启动文件
└── STM32G474XX_FLASH.ld           链接脚本
```

### 2. 当前硬件资源

| 功能 | 外设/引脚 | 当前工程配置 |
|---|---|---|
| 主 PWM 输出 | `HRTIM1_CHA1` / `PA8` | Timer A Output TA1 |
| ADC 触发源 | `HRTIM_TRG1` | Timer A Compare 4 事件 |
| ADC 采样输入 | `ADC1_IN1` / `PA0` | Regular Rank 1 |
| 触发观测脉冲 | `HRTIM1_CHB2` / `PA11` | Timer B Output TB2 |
| 状态 LED | `PC13` | ADC 转换后翻转 |
| 串口调试 | USART1/USART2 | 115200 8N1 |

其中 PA11 的 TB2 不是业务 PWM，而是调试标记输出：Timer B 被 Timer A CMP4 事件复位，然后 TB2 在 Timer B CMP1 到 CMP2 之间输出一个很窄的脉冲，方便用示波器观察 ADC 触发时刻。

## 二、PWM 与 ADC 触发时序

本工程采用 HRTIM1 Timer A 边沿对齐、向上计数。Timer A 周期为 `17000`，预分频为 `HRTIM Clock x4`。当 HRTIM 输入时钟为 170 MHz 时，Timer A 计数时钟等效为 680 MHz，因此：

```text
PWM 频率 = 680 MHz / 17000 = 40 kHz
PWM 周期 = 25 us
```

时序关系如下：

```text
Timer A  0              CMP1             CMP4                 PER
         |               |                |                    |
TA1 PWM  +---------------+________________+____________________+
         MOS 导通         MOS 关断         ADC 硬件触发
```

当前工程中：

```text
PER  = 17000
CMP1 = 5100       初始 30% 占空比
CMP4 = 11050      低电平中点，即 (CMP1 + PER) / 2
```

当占空比动态变化时，`main.c` 中的 `Boost_SetDutyPermille()` 会同时更新 CMP1 和 CMP4：

```c
cmp1 = (BOOST_HRTIM_PERIOD * duty_pm) / 1000U;

__HAL_HRTIM_SETCOMPARE(
    &hhrtim1,
    HRTIM_TIMERINDEX_TIMER_A,
    HRTIM_COMPAREUNIT_1,
    cmp1
);

__HAL_HRTIM_SETCOMPARE(
    &hhrtim1,
    HRTIM_TIMERINDEX_TIMER_A,
    HRTIM_COMPAREUNIT_4,
    (cmp1 + BOOST_HRTIM_PERIOD) / 2U
);
```

也就是说，PWM 下降沿由 CMP1 决定，ADC 触发点由 CMP4 决定。把 CMP4 放到关断区间中点，可以避开开关边沿附近的振铃和噪声。
## 三、CubeMX 详细配置步骤

下面按从零配置一个等效工程的顺序说明。CubeMX 不同版本菜单文字可能略有差异，但配置关系不变。
### 1. 新建工程

1. 打开 STM32CubeMX。
2. 在 MCU Selector 中选择 `STM32G474VETx`。
3. 创建工程后进入 Pinout & Configuration 页面。
4. 在 Project Manager 中设置：

| 项目 | 建议配置 |
|---|---|
| Project Name | `G474_HETIM_ADC` |
| Toolchain / IDE | `CMake` |
| Firmware Package | STM32Cube FW_G4 V1.6.3 或兼容版本 |
| Code Generator | 勾选保留用户代码区域 |

### 2. 配置系统时钟

进入：

```text
Clock Configuration
```

本工程最终时钟为：

| 时钟项 | 配置结果 |
|---|---|
| SYSCLK | 170 MHz |
| HCLK | 170 MHz |
| APB1 | 170 MHz |
| APB2 | 170 MHz |
| ADC12 clock | 170 MHz |
| HRTIM1 clock | 170 MHz |

当前 `main.c` 使用 HSI 作为 PLL 输入：

```text
HSI = 16 MHz
PLLM = /4
PLLN = x85
PLLR = /2
SYSCLK = 170 MHz
```

同时在系统时钟初始化中使用：

```c
HAL_PWREx_ControlVoltageScaling(PWR_REGULATOR_VOLTAGE_SCALE1_BOOST);
```

这是 STM32G4 运行到 170 MHz 时需要的电源档位配置。
![[屏幕截图 2026-07-21 190651.png]]
### 3. 配置调试接口

进入：

```text
System Core -> SYS
```

设置：

| 项目 | 配置 |
|---|---|
| Debug | Serial Wire |
| Timebase Source | SysTick |

对应引脚：

| 引脚 | 功能 |
|---|---|
| PA13 | SWDIO |
| PA14 | SWCLK |

### 4. 配置 ADC1 输入

在 Pinout 图上选择 `PA0`，设置为：

```text
ADC1_IN1
```

然后进入：

```text
Analog -> ADC1
```

基础配置：

| 参数 | 配置 |
|---|---|
| ADC mode | Independent mode |
| Resolution | 12 bits |
| Data Alignment | Right alignment |
| Scan Conversion Mode | Disabled |
| Continuous Conversion Mode | Disabled |
| Number Of Conversion | 1 |
| External Trigger Conversion Source | HRTIM_TRG1 |
| External Trigger Conversion Edge | Rising Edge |
| DMA Continuous Requests | Disabled |
| Overrun | Data preserved |
![[Pasted image 20260722025553.png]]
Regular Conversion 配置：

| 参数                  | 配置             |
| ------------------- | -------------- |
| Rank                | Regular Rank 1 |
| Channel             | ADC_CHANNEL_1  |
| Sampling Time       | 2.5 ADC cycles |
| Single/Differential | Single-ended   |
| Offset              | None           |
|                     |                |

注意：`2.5 cycles` 采样时间只适合信号源阻抗较低、前级运放驱动能力足够的情况。如果采样电阻分压或 RC 滤波阻抗较高，需要适当增大 Sampling Time，否则 ADC 采样电容可能充不满。
![[Pasted image 20260722025705.png]]

### 5. 开启 HRTIM1 的 Timer A PWM 输出

在 Pinout 图上选择 `PA8`，设置为：

```text
HRTIM1_CHA1
```

进入：

```text
Timers -> HRTIM1
```

启用：

```text
Timer A
TA1 Output
```

Timer A Time Base 配置：

| 参数 | 配置 |
|---|---|
| Basic/Advanced Configuration | Advanced |
| Prescaler Ratio | HRTIM Clock x4 |
| Period | `17000` / `0x4268` |
| Repetition Counter | 0 |
| Up Down Mode | Up-counting |
| Mode | Continuous |
| Preload Enable | Enabled |

![[Pasted image 20260722025857.png]]

### 6. 配置 Timer A Compare

在 HRTIM1 -> Timer A 下启用比较单元：

| Compare Unit | 用途 | 当前初值 |
|---|---|---|
| Compare 1 | 产生 PWM 下降沿 | `5100` |
| Compare 4 | 产生 ADC 触发事件 | `11050` |

可选保留：

| Compare Unit | 用途 | 当前初值 |
|---|---|---|
| Compare 2 | 当前未参与主触发链路 | `200` |

配置依据：

```text
CMP1 = PER * duty
CMP4 = (CMP1 + PER) / 2
```

以 30% 初始占空比为例：

```text
CMP1 = 17000 * 0.3 = 5100
CMP4 = (5100 + 17000) / 2 = 11050
```

![[Pasted image 20260722025950.png]]

### 7. 配置 TA1 输出置位/复位源

进入 HRTIM1 -> Timer A -> Output TA1 配置：

| 参数 | 配置 |
|---|---|
| Output | TA1 |
| Polarity | High |
| Set Source | Timer Period event |
| Reset Source | Timer Compare 1 event |
| Idle Mode | None |
| Idle Level | Inactive |
| Fault Level | None |
| Chopper Mode | Disabled |

这样配置后，Timer A 每个周期开始时 TA1 输出高电平，到 CMP1 时输出低电平。

![[Pasted image 20260722030042.png]]

### 8. 配置 HRTIM ADC Trigger 1

进入：

```text
Timers -> HRTIM1 -> ADC Trigger
```

配置：

| 参数 | 配置 |
|---|---|
| ADC Trigger ID | ADC Trigger 1 |
| Update Source | Timer A |
| Trigger Source | Timer A Compare 4 |
| Post Scaler | 0 |

生成代码后应能在 `Core/Src/hrtim.c` 中看到：

```c
pADCTriggerCfg.UpdateSource = HRTIM_ADCTRIGGERUPDATE_TIMER_A;
pADCTriggerCfg.Trigger = HRTIM_ADCTRIGGEREVENT13_TIMERA_CMP4;
HAL_HRTIM_ADCTriggerConfig(&hhrtim1, HRTIM_ADCTRIGGER_1, &pADCTriggerCfg);
HAL_HRTIM_ADCPostScalerConfig(&hhrtim1, HRTIM_ADCTRIGGER_1, 0);
```

这一步是整个联动关系的核心：CMP4 事件不输出到 GPIO，而是在芯片内部送到 ADC 触发矩阵。

![[Pasted image 20260722030121.png]]

### 9. 配置 ADC 外部触发源

回到：

```text
Analog -> ADC1 -> Parameter Settings
```

确认：

| 参数 | 配置 |
|---|---|
| External Trigger Conversion Source | HRTIM_TRG1 |
| External Trigger Conversion Edge | Rising Edge |
| Continuous Conversion Mode | Disabled |

生成代码后应能在 `Core/Src/adc.c` 中看到：

```c
hadc1.Init.ExternalTrigConv = ADC_EXTERNALTRIG_HRTIM_TRG1;
hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_RISING;
hadc1.Init.ContinuousConvMode = DISABLE;
```

![[Pasted image 20260722030225.png]]
### 10. 配置 Timer B 调试标记输出

如果需要用示波器确认 ADC 触发点，可以额外配置 Timer B 和 TB2。当前工程已经配置 PA11 为：

```text
HRTIM1_CHB2
```

Timer B 基础配置：

| 参数 | 配置 |
|---|---|
| Prescaler Ratio | HRTIM Clock x4 |
| Period | `17000` |
| Up Down Mode | Up-counting |
| Mode | Continuous |
| Reset Trigger | Other 1 CMP4 |
![[Pasted image 20260722030315.png]]

Timer B Compare：

| Compare Unit | 当前值 |
|---|---|
| CMP1 | `1` |
| CMP2 | `200` |

![[Pasted image 20260722030337.png]]

TB2 输出配置：

| 参数 | 配置 |
|---|---|
| Set Source | Timer Compare 1 event |
| Reset Source | Timer Compare 2 event |

其效果是：Timer A CMP4 发生时复位 Timer B；Timer B 随后从 0 开始计数，在 CMP1 置高，在 CMP2 拉低，于是 PA11 输出一个位于 ADC 触发点附近的窄脉冲。这个脉冲便于观察，不参与 ADC 触发本身。

![[Pasted image 20260722030402.png]]

### 11. 配置 GPIO 和串口调试

PC13 配置为普通输出，用于在收到 ADC 转换结果后翻转 LED：

```text
PC13 -> GPIO_Output
User Label -> User_led
```

USART 可按板卡实际引脚配置。当前源码使用 USART1 和 USART2 输出调试字符串：

```text
BOOT UART1+UART2 115200 8N1
BOOST:ADC_CAL
BOOST:HRTIM_UPDATE
BOOST:ADC_START
BOOST:PWM_OUTPUT
BOOST:PWM_COUNTER
BOOST:OK
ADC=xxxx
```

如果只使用一个串口，需要同步修改 `Debug_Write()`，避免访问未初始化的串口句柄。

## 四、生成代码后的关键检查

### 1. HRTIM 初始化

`Core/Src/hrtim.c` 中应具备以下关键配置：

```c
pTimeBaseCfg.Period = 0x4268;
pTimeBaseCfg.PrescalerRatio = HRTIM_PRESCALERRATIO_MUL4;
pTimerCtl.UpDownMode = HRTIM_TIMERUPDOWNMODE_UP;

pCompareCfg.CompareValue = 5100;
HAL_HRTIM_WaveformCompareConfig(&hhrtim1,
                                HRTIM_TIMERINDEX_TIMER_A,
                                HRTIM_COMPAREUNIT_1,
                                &pCompareCfg);

pCompareCfg.CompareValue = 11050;
HAL_HRTIM_WaveformCompareConfig(&hhrtim1,
                                HRTIM_TIMERINDEX_TIMER_A,
                                HRTIM_COMPAREUNIT_4,
                                &pCompareCfg);

pOutputCfg.SetSource = HRTIM_OUTPUTSET_TIMPER;
pOutputCfg.ResetSource = HRTIM_OUTPUTRESET_TIMCMP1;
```

### 2. ADC 初始化

`Core/Src/adc.c` 中应具备：

```c
hadc1.Init.ContinuousConvMode = DISABLE;
hadc1.Init.NbrOfConversion = 1;
hadc1.Init.ExternalTrigConv = ADC_EXTERNALTRIG_HRTIM_TRG1;
hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_RISING;

sConfig.Channel = ADC_CHANNEL_1;
sConfig.Rank = ADC_REGULAR_RANK_1;
sConfig.SamplingTime = ADC_SAMPLETIME_2CYCLES_5;
```

### 3. 启动顺序

`Core/Src/main.c` 中 `Boost_Start()` 的启动顺序很重要：

```text
ADC 校准
设置初始 CMP1/CMP4
HRTIM Software Update，让预装载值进入活动寄存器
HAL_ADC_Start()，让 ADC 等待外部触发
HAL_HRTIM_WaveformOutputStart()，打开 TA1/TB2 输出
HAL_HRTIM_WaveformCounterStart()，启动 Timer A/Timer B 计数器
```

对应代码：

```c
HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED);
Boost_SetDutyPermille(BOOST_DUTY_MIN_PM);

HAL_HRTIM_SoftwareUpdate(&hhrtim1,
                         HRTIM_TIMERUPDATE_A | HRTIM_TIMERUPDATE_B);

HAL_ADC_Start(&hadc1);

HAL_HRTIM_WaveformOutputStart(&hhrtim1,
                              HRTIM_OUTPUT_TA1 | HRTIM_OUTPUT_TB2);

HAL_HRTIM_WaveformCounterStart(&hhrtim1,
                               HRTIM_TIMERID_TIMER_A | HRTIM_TIMERID_TIMER_B);
```

ADC 启动后并不会立刻转换，而是等待 HRTIM_TRG1。每次 CMP4 事件到来，ADC Regular 组完成一次转换。

## 五、main.c 用户代码详解

`main.c` 可以分成两类代码：

1. CubeMX 生成代码：`SystemClock_Config()`、`MX_GPIO_Init()`、`MX_HRTIM1_Init()`、`MX_ADC1_Init()`、`MX_USARTx_UART_Init()` 等。
2. 用户自己写的应用代码：Boost PWM 启动、动态占空比更新、ADC 结果读取、串口调试输出。

下面重点讲第二类，也就是移植或复现时最需要关心的部分。

### 1. 用户全局变量

```c
volatile uint16_t g_vout_adc_raw = 0;
volatile uint8_t  g_adc_new_flag = 0;
```

`g_vout_adc_raw` 保存最近一次 ADC 原始采样值。ADC 当前配置为 12 位分辨率，所以理论范围是 `0 ~ 4095`。

`g_adc_new_flag` 用来标记“主循环刚拿到了一次新的 ADC 数据”。当前工程只是置 1，还没有在别处消费它；如果后续要加闭环控制、串口上报状态机或滤波处理，可以用这个标志作为新数据提示。

这里使用 `volatile`，是因为这些变量可能在主循环和中断回调之间共享。虽然当前主循环主要用轮询方式读取 ADC，但文件末尾也保留了 `HAL_ADC_ConvCpltCallback()`，因此用 `volatile` 更稳妥。

### 2. Boost 相关宏定义

```c
#define BOOST_HRTIM_PERIOD   17000U
#define BOOST_DUTY_MIN_PM    300U    // 30.0%
#define BOOST_DUTY_MAX_PM    700U    // 70.0%
#define BOOST_DUTY_STEP_PM   10U     // 1.0%
#define BOOST_DUTY_STEP_MS   20U
```

这些宏把 PWM 的几个关键参数集中放在一起：

| 宏 | 含义 |
|---|---|
| `BOOST_HRTIM_PERIOD` | HRTIM Timer A 周期计数值，对应前文的 `PER = 17000` |
| `BOOST_DUTY_MIN_PM` | 最小占空比，单位是千分比，`300` 表示 30.0% |
| `BOOST_DUTY_MAX_PM` | 最大占空比，`700` 表示 70.0% |
| `BOOST_DUTY_STEP_PM` | 每次改变的占空比步进，`10` 表示 1.0% |
| `BOOST_DUTY_STEP_MS` | 每隔多少毫秒更新一次占空比 |

这里没有用浮点数，而是用“千分比 permille”。这样做的好处是计算简单、执行快，并且在 Cortex-M 上不依赖浮点格式转换。比如：

```text
duty_pm = 300  -> 30.0%
duty_pm = 510  -> 51.0%
duty_pm = 700  -> 70.0%
```

### 3. main() 的执行流程

`main()` 的核心流程如下：

```c
HAL_Init();
SystemClock_Config();

MX_GPIO_Init();
MX_HRTIM1_Init();
MX_ADC1_Init();
MX_USART1_UART_Init();
MX_USART2_UART_Init();

Debug_PrintBoot();
Boost_Start();

while (1)
{
  Boost_UpdateDutySweep();

  if (HAL_ADC_PollForConversion(&hadc1, 0) == HAL_OK)
  {
    g_vout_adc_raw = HAL_ADC_GetValue(&hadc1);
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
    g_adc_new_flag = 1;

    Debug_PrintAdcRaw(g_vout_adc_raw);

    HAL_ADC_Stop(&hadc1);
    HAL_ADC_Start(&hadc1);
  }
  else
  {
    Debug_PrintWaitAdc();
  }
}
```

启动阶段先由 HAL 和 CubeMX 初始化系统时钟、GPIO、HRTIM、ADC、串口。注意顺序上需要先初始化外设，再调用用户自己的 `Boost_Start()`，因为 `Boost_Start()` 内部会直接访问 `hadc1`、`hhrtim1`、`huart1` 和 `huart2`。

主循环中做两件事：

1. 调用 `Boost_UpdateDutySweep()`，按 20 ms 周期更新 PWM 占空比。
2. 轮询 ADC 是否已经被 HRTIM_TRG1 触发并完成转换。

这里 `HAL_ADC_PollForConversion(&hadc1, 0)` 的超时时间是 `0`，表示不阻塞等待。如果 ADC 已经转换完成，就立即返回 `HAL_OK`；如果还没完成，主循环继续往下走，并通过 `Debug_PrintWaitAdc()` 低频输出等待信息。这样不会因为串口或 ADC 等待把主循环卡死。

### 4. Boost_SetDutyPermille()

`Boost_SetDutyPermille()` 的作用是：根据目标占空比同时更新 Timer A 的 `CMP1` 和 `CMP4`。

完整函数如下：

```c
static void Boost_SetDutyPermille(uint16_t duty_pm)
{
  uint32_t cmp1;

  if (duty_pm < BOOST_DUTY_MIN_PM)
  {
    duty_pm = BOOST_DUTY_MIN_PM;
  }

  if (duty_pm > BOOST_DUTY_MAX_PM)
  {
    duty_pm = BOOST_DUTY_MAX_PM;
  }

  cmp1 = (BOOST_HRTIM_PERIOD * duty_pm) / 1000U;

  __HAL_HRTIM_SETCOMPARE(&hhrtim1,
                         HRTIM_TIMERINDEX_TIMER_A,
                         HRTIM_COMPAREUNIT_1,
                         cmp1);

  __HAL_HRTIM_SETCOMPARE(&hhrtim1,
                         HRTIM_TIMERINDEX_TIMER_A,
                         HRTIM_COMPAREUNIT_4,
                         (cmp1 + BOOST_HRTIM_PERIOD) / 2U);
}
```

函数开始先做限幅。如果传进来的占空比小于 30%，就强制变成 30%；如果大于 70%，就强制变成 70%。这一步很重要，尤其是在 Boost 电路里，占空比过高可能让电感电流、输出电压或功率器件压力超出预期。

然后计算 `CMP1`：

```text
CMP1 = PERIOD * duty_pm / 1000
```

例如 `PERIOD = 17000`：

| 占空比 | `duty_pm` | `CMP1` |
|---|---:|---:|
| 30.0% | 300 | 5100 |
| 50.0% | 500 | 8500 |
| 70.0% | 700 | 11900 |

Timer A 的 TA1 输出配置是：

```text
Period event       -> TA1 置高
Compare 1 event    -> TA1 拉低
```

所以 `CMP1` 越大，高电平时间越长，占空比越大。

接着计算 `CMP4`：

```text
CMP4 = (CMP1 + PERIOD) / 2
```

因为本工程希望 ADC 在 PWM 低电平区间中间采样，而低电平区间是：

```text
CMP1 -> PERIOD
```

所以这个区间的中点就是 `(CMP1 + PERIOD) / 2`。例如 30% 占空比时：

```text
CMP1 = 5100
CMP4 = (5100 + 17000) / 2 = 11050
```

这样做的效果是：占空比变化时，ADC 触发点不会固定死在某个计数值，而是始终跟着 PWM 关断区间移动。对于 Boost 来说，这通常比贴着开关边沿采样更干净。

### 5. Boost_UpdateDutySweep()

`Boost_UpdateDutySweep()` 是演示用的占空比扫描函数。它让 PWM 占空比在 30% 和 70% 之间来回变化，方便用示波器看到 PWM 宽度和 ADC 触发点一起移动。

完整函数如下：

```c
static void Boost_UpdateDutySweep(void)
{
  static uint32_t last_update_ms = 0U;
  static uint16_t duty_pm = BOOST_DUTY_MIN_PM;
  static int8_t direction = 1;
  uint32_t now_ms = HAL_GetTick();

  if ((now_ms - last_update_ms) < BOOST_DUTY_STEP_MS)
  {
    return;
  }
  last_update_ms = now_ms;

  if (direction > 0)
  {
    if ((uint16_t)(duty_pm + BOOST_DUTY_STEP_PM) >= BOOST_DUTY_MAX_PM)
    {
      duty_pm = BOOST_DUTY_MAX_PM;
      direction = -1;
    }
    else
    {
      duty_pm += BOOST_DUTY_STEP_PM;
    }
  }
  else
  {
    if (duty_pm <= (uint16_t)(BOOST_DUTY_MIN_PM + BOOST_DUTY_STEP_PM))
    {
      duty_pm = BOOST_DUTY_MIN_PM;
      direction = 1;
    }
    else
    {
      duty_pm -= BOOST_DUTY_STEP_PM;
    }
  }

  Boost_SetDutyPermille(duty_pm);
}
```

函数内部有 3 个 `static` 局部变量：

| 变量 | 含义 |
|---|---|
| `last_update_ms` | 上一次更新占空比时的系统 tick |
| `duty_pm` | 当前占空比，单位是千分比 |
| `direction` | 扫描方向，`1` 表示往 70% 增加，`-1` 表示往 30% 减小 |

第一段代码用于控制更新周期：

```c
uint32_t now_ms = HAL_GetTick();

if ((now_ms - last_update_ms) < BOOST_DUTY_STEP_MS)
{
  return;
}
last_update_ms = now_ms;
```

`HAL_GetTick()` 默认返回系统启动后的毫秒计数。这里用 `now_ms - last_update_ms` 判断是否已经过去 20 ms。时间没到就直接 `return`，不更新占空比。这个写法还有一个好处：即使 `HAL_GetTick()` 的 32 位计数溢出， unsigned 减法也能正常工作。

第二段处理“向上扫”：

```c
if (direction > 0)
{
  if ((uint16_t)(duty_pm + BOOST_DUTY_STEP_PM) >= BOOST_DUTY_MAX_PM)
  {
    duty_pm = BOOST_DUTY_MAX_PM;
    direction = -1;
  }
  else
  {
    duty_pm += BOOST_DUTY_STEP_PM;
  }
}
```

当方向为正时，每次增加 `BOOST_DUTY_STEP_PM`，也就是 1%。如果下一步已经到达或超过 70%，就直接钳到 70%，并把方向改成 `-1`，下一次开始往下扫。

第三段处理“向下扫”：

```c
else
{
  if (duty_pm <= (uint16_t)(BOOST_DUTY_MIN_PM + BOOST_DUTY_STEP_PM))
  {
    duty_pm = BOOST_DUTY_MIN_PM;
    direction = 1;
  }
  else
  {
    duty_pm -= BOOST_DUTY_STEP_PM;
  }
}
```

向下扫时每次减 1%。如果已经接近 30%，就直接钳到 30%，并把方向改回 `1`。

最后调用：

```c
Boost_SetDutyPermille(duty_pm);
```

这一句才是真正写入 HRTIM 比较寄存器的地方。也就是说，`Boost_UpdateDutySweep()` 只负责决定“下一个占空比是多少”，而 `Boost_SetDutyPermille()` 负责把占空比转换成 `CMP1` 和 `CMP4`。

按当前参数计算，30% 到 70% 一共相差 40%，每次走 1%，每步 20 ms，所以从 30% 扫到 70% 大约需要：

```text
40 * 20 ms = 800 ms
```

再从 70% 回到 30% 也是约 800 ms，一个完整往返约 1.6 s。

### 6. Boost_Start()

`Boost_Start()` 是 PWM 和 ADC 联动启动函数。它必须在 CubeMX 初始化完 HRTIM、ADC、USART 之后调用。

启动顺序如下：

```text
串口打印 BOOST:ADC_CAL
ADC 自校准
设置初始占空比 30%，同时设置 CMP1/CMP4
触发 HRTIM Software Update
启动 ADC，让 ADC 等待 HRTIM_TRG1
打开 TA1 和 TB2 输出
启动 Timer A 和 Timer B 计数器
串口打印 BOOST:OK
```

其中最关键的是 `HAL_ADC_Start()` 和 `HAL_HRTIM_WaveformCounterStart()` 的顺序。这里先启动 ADC，再启动 HRTIM 计数器，是为了让 ADC 先进入“等待外部触发”的状态。随后 Timer A 开始运行，到了 CMP4 时，HRTIM_TRG1 才能触发 ADC Regular 转换。

如果反过来先启动 HRTIM，再启动 ADC，前几个 HRTIM_TRG1 可能已经发生但 ADC 还没准备好，调试时容易误判为触发链路不稳定。

### 7. 主循环读取 ADC

当前工程在主循环里轮询 ADC 转换完成：

```c
if (HAL_ADC_PollForConversion(&hadc1, 0) == HAL_OK)
{
  g_vout_adc_raw = HAL_ADC_GetValue(&hadc1);
  HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
  g_adc_new_flag = 1;

  Debug_PrintAdcRaw(g_vout_adc_raw);

  if (HAL_ADC_Stop(&hadc1) != HAL_OK)
  {
    Debug_PrintAdcRestartFail();
  }

  if (HAL_ADC_Start(&hadc1) != HAL_OK)
  {
    Debug_PrintAdcRestartFail();
  }
}
else
{
  Debug_PrintWaitAdc();
}
```

读取顺序是：

1. `HAL_ADC_PollForConversion(&hadc1, 0)` 检查 ADC 是否转换完成。
2. `HAL_ADC_GetValue(&hadc1)` 读取 ADC 数据寄存器。
3. `HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13)` 翻转 LED，作为“拿到一次采样”的可见标记。
4. `Debug_PrintAdcRaw()` 每 100 ms 打印一次 ADC 原始值。
5. `HAL_ADC_Stop()` 后再 `HAL_ADC_Start()`，让 ADC Regular 组重新等待下一次 HRTIM_TRG1。

这里使用 `Stop + Start` 是为了演示逻辑直观。对于更高频、更实时的控制系统，可以进一步改成：

| 方案 | 适用场景 |
|---|---|
| ADC interrupt | 中等频率采样，需要转换完成回调 |
| ADC DMA circular | 连续采样缓存，CPU 负担低 |
| ADC injected group | 电源控制中更常用，可与 PWM 触发强绑定 |

当前文档描述的是本工程已实现的 Regular + HRTIM_TRG1 轮询方式。

### 8. 串口调试函数

`Debug_Write()` 是最底层的串口发送函数：

```c
static void Debug_Write(const uint8_t *msg, uint16_t len)
{
  HAL_UART_Transmit(&huart1, (uint8_t *)msg, len, 100U);
  HAL_UART_Transmit(&huart2, (uint8_t *)msg, len, 100U);
}
```

它会把同一条调试信息同时发到 USART1 和 USART2。这样做适合调试阶段：不确定板子上哪个串口接到了 USB 转串口时，两个口同时输出更省事。正式项目里可以只保留一个串口，或者改成 DMA/环形缓冲发送，避免阻塞。

`Debug_PrintAdcRaw()` 用于低频打印 ADC 值：

```c
static void Debug_PrintAdcRaw(uint16_t adc_raw)
{
  static uint32_t last_print_ms = 0U;
  char tx_buf[24];
  uint32_t now_ms = HAL_GetTick();

  if ((last_print_ms != 0U) && ((now_ms - last_print_ms) < 100U))
  {
    return;
  }
  last_print_ms = now_ms;

  int tx_len = snprintf(tx_buf, sizeof(tx_buf), "ADC=%u\r\n", (unsigned int)adc_raw);
  if (tx_len > 0)
  {
    Debug_Write((uint8_t *)tx_buf, (uint16_t)tx_len);
  }
}
```

虽然 PWM 是 40 kHz，ADC 理论上可以非常频繁地被触发，但串口打印不能按 40 kHz 输出。115200 波特率下，打印太快会严重阻塞主循环。所以这里限制为每 100 ms 打印一次，也就是约 10 Hz。

`Debug_PrintWaitAdc()` 类似，只是等待 ADC 时每 500 ms 输出一次 `WAIT_ADC`，避免串口被刷屏：

```c
static void Debug_PrintWaitAdc(void)
{
  static uint32_t last_print_ms = 0U;
  static const uint8_t wait_msg[] = "WAIT_ADC\r\n";
  uint32_t now_ms = HAL_GetTick();

  if ((now_ms - last_print_ms) < 500U)
  {
    return;
  }
  last_print_ms = now_ms;

  Debug_Write(wait_msg, sizeof(wait_msg) - 1U);
}
```

`Debug_PrintBoot()` 和 `Debug_PrintAdcRestartFail()` 则分别用于上电提示和 ADC 重启失败提示。

### 9. HAL_ADC_ConvCpltCallback()

文件末尾还保留了一个 ADC 转换完成回调：

```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
  if (hadc->Instance == ADC1)
  {
    GPIOC->BSRR = GPIO_PIN_13;
    g_vout_adc_raw = HAL_ADC_GetValue(hadc);
    GPIOC->BRR = GPIO_PIN_13;
  }
}
```

这个函数只有在使用 `HAL_ADC_Start_IT()` 这类中断方式启动 ADC 时才会被 HAL 自动调用。当前主循环里用的是 `HAL_ADC_Start()` + `HAL_ADC_PollForConversion()`，所以主要路径不会依赖这个回调。

保留它的意义是：后续如果把 ADC 读取方式改成中断模式，可以直接把采样处理搬到这里。`GPIOC->BSRR` 和 `GPIOC->BRR` 是直接寄存器操作，比 `HAL_GPIO_WritePin()` 更快，适合在中断里打一个很窄的测量脉冲。

## 六、完整 main.c 参考代码

下面这份代码保留了本文讲解的用户逻辑。CubeMX 生成的外设初始化函数只保留调用和声明，不展开 `SystemClock_Config()`、`MX_HRTIM1_Init()`、`MX_ADC1_Init()` 等函数体；这些函数仍然由 CubeMX 在对应文件中生成。

```c
#include "main.h"
#include "adc.h"
#include "hrtim.h"
#include "usart.h"
#include "gpio.h"

#include <stdio.h>

volatile uint16_t g_vout_adc_raw = 0;
volatile uint8_t  g_adc_new_flag = 0;

#define BOOST_HRTIM_PERIOD   17000U
#define BOOST_DUTY_MIN_PM    300U
#define BOOST_DUTY_MAX_PM    700U
#define BOOST_DUTY_STEP_PM   10U
#define BOOST_DUTY_STEP_MS   20U

void SystemClock_Config(void);

static void Boost_SetDutyPermille(uint16_t duty_pm);
static void Boost_UpdateDutySweep(void);
static void Boost_Start(void);
static void Debug_Write(const uint8_t *msg, uint16_t len);
static void Debug_PrintAdcRaw(uint16_t adc_raw);
static void Debug_PrintAdcRestartFail(void);
static void Debug_PrintBoot(void);
static void Debug_PrintWaitAdc(void);

int main(void)
{
  HAL_Init();
  SystemClock_Config();

  MX_GPIO_Init();
  MX_HRTIM1_Init();
  MX_ADC1_Init();
  MX_USART1_UART_Init();
  MX_USART2_UART_Init();

  Debug_PrintBoot();
  Boost_Start();

  while (1)
  {
    Boost_UpdateDutySweep();

    if (HAL_ADC_PollForConversion(&hadc1, 0) == HAL_OK)
    {
      g_vout_adc_raw = HAL_ADC_GetValue(&hadc1);
      HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
      g_adc_new_flag = 1;

      Debug_PrintAdcRaw(g_vout_adc_raw);

      if (HAL_ADC_Stop(&hadc1) != HAL_OK)
      {
        Debug_PrintAdcRestartFail();
      }

      if (HAL_ADC_Start(&hadc1) != HAL_OK)
      {
        Debug_PrintAdcRestartFail();
      }
    }
    else
    {
      Debug_PrintWaitAdc();
    }
  }
}

static void Boost_SetDutyPermille(uint16_t duty_pm)
{
  uint32_t cmp1;

  if (duty_pm < BOOST_DUTY_MIN_PM)
  {
    duty_pm = BOOST_DUTY_MIN_PM;
  }

  if (duty_pm > BOOST_DUTY_MAX_PM)
  {
    duty_pm = BOOST_DUTY_MAX_PM;
  }

  cmp1 = (BOOST_HRTIM_PERIOD * duty_pm) / 1000U;

  __HAL_HRTIM_SETCOMPARE(&hhrtim1,
                         HRTIM_TIMERINDEX_TIMER_A,
                         HRTIM_COMPAREUNIT_1,
                         cmp1);

  __HAL_HRTIM_SETCOMPARE(&hhrtim1,
                         HRTIM_TIMERINDEX_TIMER_A,
                         HRTIM_COMPAREUNIT_4,
                         (cmp1 + BOOST_HRTIM_PERIOD) / 2U);
}

static void Boost_UpdateDutySweep(void)
{
  static uint32_t last_update_ms = 0U;
  static uint16_t duty_pm = BOOST_DUTY_MIN_PM;
  static int8_t direction = 1;
  uint32_t now_ms = HAL_GetTick();

  if ((now_ms - last_update_ms) < BOOST_DUTY_STEP_MS)
  {
    return;
  }
  last_update_ms = now_ms;

  if (direction > 0)
  {
    if ((uint16_t)(duty_pm + BOOST_DUTY_STEP_PM) >= BOOST_DUTY_MAX_PM)
    {
      duty_pm = BOOST_DUTY_MAX_PM;
      direction = -1;
    }
    else
    {
      duty_pm += BOOST_DUTY_STEP_PM;
    }
  }
  else
  {
    if (duty_pm <= (uint16_t)(BOOST_DUTY_MIN_PM + BOOST_DUTY_STEP_PM))
    {
      duty_pm = BOOST_DUTY_MIN_PM;
      direction = 1;
    }
    else
    {
      duty_pm -= BOOST_DUTY_STEP_PM;
    }
  }

  Boost_SetDutyPermille(duty_pm);
}

static void Boost_Start(void)
{
  static const uint8_t msg_cal[] = "BOOST:ADC_CAL\r\n";
  static const uint8_t msg_update[] = "BOOST:HRTIM_UPDATE\r\n";
  static const uint8_t msg_adc_start[] = "BOOST:ADC_START\r\n";
  static const uint8_t msg_pwm_out[] = "BOOST:PWM_OUTPUT\r\n";
  static const uint8_t msg_pwm_cnt[] = "BOOST:PWM_COUNTER\r\n";
  static const uint8_t msg_ok[] = "BOOST:OK\r\n";

  Debug_Write(msg_cal, sizeof(msg_cal) - 1U);
  if (HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED) != HAL_OK)
  {
    Error_Handler();
  }

  Boost_SetDutyPermille(BOOST_DUTY_MIN_PM);

  Debug_Write(msg_update, sizeof(msg_update) - 1U);
  if (HAL_HRTIM_SoftwareUpdate(&hhrtim1,
                               HRTIM_TIMERUPDATE_A | HRTIM_TIMERUPDATE_B) != HAL_OK)
  {
    Error_Handler();
  }

  Debug_Write(msg_adc_start, sizeof(msg_adc_start) - 1U);
  if (HAL_ADC_Start(&hadc1) != HAL_OK)
  {
    Error_Handler();
  }

  Debug_Write(msg_pwm_out, sizeof(msg_pwm_out) - 1U);
  if (HAL_HRTIM_WaveformOutputStart(&hhrtim1,
                                    HRTIM_OUTPUT_TA1 | HRTIM_OUTPUT_TB2) != HAL_OK)
  {
    Error_Handler();
  }

  Debug_Write(msg_pwm_cnt, sizeof(msg_pwm_cnt) - 1U);
  if (HAL_HRTIM_WaveformCounterStart(&hhrtim1,
                                     HRTIM_TIMERID_TIMER_A | HRTIM_TIMERID_TIMER_B) != HAL_OK)
  {
    Error_Handler();
  }

  Debug_Write(msg_ok, sizeof(msg_ok) - 1U);
}

static void Debug_Write(const uint8_t *msg, uint16_t len)
{
  HAL_UART_Transmit(&huart1, (uint8_t *)msg, len, 100U);
  HAL_UART_Transmit(&huart2, (uint8_t *)msg, len, 100U);
}

static void Debug_PrintAdcRaw(uint16_t adc_raw)
{
  static uint32_t last_print_ms = 0U;
  char tx_buf[24];
  uint32_t now_ms = HAL_GetTick();

  if ((last_print_ms != 0U) && ((now_ms - last_print_ms) < 100U))
  {
    return;
  }
  last_print_ms = now_ms;

  int tx_len = snprintf(tx_buf, sizeof(tx_buf), "ADC=%u\r\n", (unsigned int)adc_raw);
  if (tx_len > 0)
  {
    Debug_Write((uint8_t *)tx_buf, (uint16_t)tx_len);
  }
}

static void Debug_PrintAdcRestartFail(void)
{
  static const uint8_t fail_msg[] = "ADC_RESTART_FAIL\r\n";

  Debug_Write(fail_msg, sizeof(fail_msg) - 1U);
}

static void Debug_PrintBoot(void)
{
  static const uint8_t boot_msg[] = "BOOT UART1+UART2 115200 8N1\r\n";

  Debug_Write(boot_msg, sizeof(boot_msg) - 1U);
}

static void Debug_PrintWaitAdc(void)
{
  static uint32_t last_print_ms = 0U;
  static const uint8_t wait_msg[] = "WAIT_ADC\r\n";
  uint32_t now_ms = HAL_GetTick();

  if ((now_ms - last_print_ms) < 500U)
  {
    return;
  }
  last_print_ms = now_ms;

  Debug_Write(wait_msg, sizeof(wait_msg) - 1U);
}

void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
  if (hadc->Instance == ADC1)
  {
    GPIOC->BSRR = GPIO_PIN_13;
    g_vout_adc_raw = HAL_ADC_GetValue(hadc);
    GPIOC->BRR = GPIO_PIN_13;
  }
}
```

## 七、编译与下载

本工程是 CMake 工程，可使用预设构建：

```powershell
cmake --preset Debug
cmake --build --preset Debug
```

生成产物通常位于：

```text
build/Debug/G474_HETIM_ADC.elf
```

下载方式可按自己的工具链选择，例如 STM32CubeProgrammer、ST-LINK、OpenOCD 或 PlatformIO 工具。

## 八、上板验证步骤

1. 连接示波器到 PA8，确认 TA1 PWM 输出。
2. PWM 频率应约为 40 kHz。
3. 占空比会在 30% 到 70% 之间缓慢往返变化。
4. 连接示波器到 PA11，确认 TB2 窄脉冲位于 PWM 低电平区间中间附近。
5. 串口输出应出现 `BOOST:OK` 和周期性的 `ADC=xxxx`。
6. PC13 应在 ADC 转换完成后翻转。
7. 改变 PA0 输入电压，串口 ADC 数值应随之变化。


![[cf0ab5e50b381fe6fb36af5306f6d133.jpg]]


## 九、常见问题

### 1. PA8 没有 PWM 输出

检查：

```text
是否调用 MX_HRTIM1_Init()
是否调用 HAL_HRTIM_WaveformOutputStart()
是否调用 HAL_HRTIM_WaveformCounterStart()
PA8 是否配置为 HRTIM1_CHA1 / AF13
```

### 2. ADC 一直没有数据

检查：

```text
ADC 是否配置为 HRTIM_TRG1 Rising Edge
HRTIM ADC Trigger 1 是否来自 Timer A Compare 4
HAL_ADC_Start() 是否在 HRTIM 计数器启动前调用
CMP4 是否小于 Period，且位于有效计数范围内
```

### 3. ADC 数据抖动很大

优先检查硬件和采样点：

```text
采样点是否离开开关边沿足够远
PA0 前端源阻抗是否过高
Sampling Time 是否过短
模拟地和功率地布局是否合适
ADC 输入是否有合适的 RC 滤波
```

### 4. 修改占空比后采样点不对

确认更新占空比时同时更新 CMP4：

```c
CMP4 = (CMP1 + PERIOD) / 2;
```

如果只更新 CMP1，不更新 CMP4，ADC 采样点就不会跟随 PWM 低电平区间移动。

## 十、关键结论

本工程的核心链路是：

```text
Timer A Period  -> TA1 置高
Timer A CMP1    -> TA1 拉低，形成 PWM 占空比
Timer A CMP4    -> HRTIM ADC Trigger 1
HRTIM_TRG1      -> ADC1 Regular Conversion
```

这种方式让 PWM 边沿和 ADC 采样触发都由 HRTIM 硬件直接产生，采样时刻确定、抖动小，不依赖 CPU 中断响应时间。对于功率电子控制，这比“PWM 中断里启动 ADC”更可靠。
