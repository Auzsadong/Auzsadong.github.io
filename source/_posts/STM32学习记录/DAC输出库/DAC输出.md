---
title: "STM32开发备忘录：TIM触发DAC与底层驱动指南"
date: 2026-05-31 16:30:00
math: true
categories:
  - [嵌入式开发, STM32]
tags: [DAC, TIM, DMA, 笔记, 嵌入式开发]
---

# STM32开发备忘录：TIM触发DAC与底层驱动指南

在嵌入式信号处理中，如果仅依靠 CPU 软件循环或延时来刷新 DAC 的输出寄存器，不仅极大地占用系统资源，而且波形输出极易受到系统中断的干扰，产生时序抖动（Jitter）。

本备忘录记录了**“定时器（TIM）触发 + DMA 自动搬运 + DAC 硬件输出”**的标准配置流程。通过配置纯硬件链路，可以彻底解放 CPU，实现精准控制输出频率以及极其平滑的任意波形生成。

---

## 第一步：配置基本定时器 (TIM6)

基本定时器（如 TIM6 / TIM7）非常适合用作内部硬件触发源。

1. **启用定时器**：在 STM32CubeMX 左侧菜单栏选择 **Timers -> TIM6**，勾选 **Activated** 开启外设。
2. **配置频率参数**：在下方的 **Configuration -> Parameter Settings** 中设置预分频器和重装载值：
   - **Prescaler (PSC)**：假设系统时钟 ($f_{\text{TIM\_CLK}}$) 为 170MHz，设为 `170 - 1`，则分频后的定时器计数基准时钟为 1MHz。
   - **Counter Period (ARR)**：假设希望 DAC 的触发采样率为 10kHz，则设为 `100 - 1`。
   
   > **触发频率计算公式：**
   > $$f_{\text{trigger}} = \frac{f_{\text{TIM\_CLK}}}{(PSC + 1) \times (ARR + 1)}$$

3. **设置触发输出 (TRGO)【关键】**：
   - 展开 **Trigger Output (TRGO) Parameters**。
   - 将 **Trigger Event Selection** 设置为 **Update Event**。这表示每次定时器计数溢出（Update）时，硬件都会在芯片内部总线上广播一个触发信号。

![TIM6触发配置](./TIM6触发配置.png)

---

## 第二步：配置数模转换器 (DAC)

1. **选择通道**：在左侧菜单栏选择 **Analog -> DAC1**。
2. **配置输出引脚网络**：在 **OUT1 Configuration** 中：
   - 若仅需输出到外部物理引脚，勾选 **Connected to external pin only**。
   - 若内部需要级联比较器或运放等外设联调，选择 **Connected to external pin and to on chip peripherals**。
   
   > 💡 **硬件避坑提示：**
   > 关于 **Output Buffer**：启用该缓冲器可以大幅增强引脚的输出驱动能力，降低输出阻抗。但是，如果你的硬件开发板上 DAC 输出端已经级联了外部运放（Op-Amp）构成的跟随器或滤波电路，建议将其设为 **Disable**，以避免内部缓冲器带来的额外失调电压影响精度。

3. **配置触发参数 (Parameter Settings)**：
   - **Trigger**：选择 **Timer 6 Trigger Out event**（必须与第一步配置的触发源严格对应）。
   - **Wave generation mode**：设置为 **Disabled**（本文采用自定义波形数组，不使用内置的三角波/噪声波生成器）。

![DAC配置参数](./DAC配置参数.png)

---

## 第三步：配置直接内存访问 (DMA)

为了让数据源源不断地送入 DAC，我们需要引入 DMA。

1. **添加通道**：在 DAC 的配置页面，切换到 **DMA Settings** 标签页。
2. **绑定请求**：点击 **Add**，分配 `DAC1_CH1` 请求。
3. **模式与数据流设置**：
   - **Mode**：强烈建议设置为 **Circular**（循环模式）。这样当 DMA 搬运完数组的最后一个数据后，会自动回到数组头部重新开始，实现周期波形的无限续航。
   - **Data Width**：Peripheral（外设）与 Memory（内存）均设置为 **Word**（32位宽，适配 STM32 的总线和 12-bit 数据对齐）。

![DMA配置页面](./DMA配置页面.png)

---

## 第四步：核心驱动代码实现

在 STM32CubeMX 生成代码并完成初始化后，可在业务逻辑层添加以下波形查找表（LUT）与启动代码。这里以一个 64 个采样点的完整正弦波为例（数据范围 0~4095，适配 12-bit DAC）：

```c
/* USER CODE BEGIN 0 */

// 定义64点的 12-bit 正弦波数据表
const uint32_t wave_data_64[64] = {  
    2048, 2248, 2447, 2642, 2831, 3013, 3185, 3346,  
    3495, 3630, 3750, 3853, 3939, 4006, 4056, 4085,  
    4095, 4085, 4056, 4006, 3939, 3853, 3750, 3630,  
    3495, 3346, 3185, 3013, 2831, 2642, 2447, 2248,  
    2048, 1847, 1648, 1453, 1264, 1082,  910,  749,  
     600,  465,  345,  242,  156,   89,   39,   10,  
       0,   10,   39,   89,  156,  242,  345,  465,  
     600,  749,  910, 1082, 1264, 1453, 1648, 1847 
};

/**
 * @brief  启动 DAC 波形输出
 * @note   务必确保 DMA 和 TIM 在此之前已完成系统初始化
 */
void DAC_Wave_Start(void)
{
    // 1. 开启 DAC 的 DMA 循环搬运
    // 参数含义：DAC句柄, 通道, 数据表首地址(强转为uint32_t*), 采样点数量, 12位右对齐模式
    HAL_DAC_Start_DMA(&hdac1, DAC_CHANNEL_1, (uint32_t*)wave_data_64, 64, DAC_ALIGN_12B_R);  
      
    // 2. 启动 TIM6 定时器，开始向 DAC 发送触发脉冲
    HAL_TIM_Base_Start(&htim6);
}

/* USER CODE END 0 */