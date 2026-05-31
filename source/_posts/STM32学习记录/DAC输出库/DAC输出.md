---
title: "STM32开发备忘录：TIM触发DAC与底层驱动指南"
date: 2026-04-11 16:30:00
math: true
categories:
  - [嵌入式开发, STM32]
tags: [DAC, TIM, DMA, 笔记, 嵌入式开发]
---

# STM32开发备忘录：TIM触发DAC与底层驱动指南

本备忘录记录了使用定时器（TIM）触发数模转换器（DAC）输出的配置流程。通过定时器触发可以精准控制DAC的输出频率以及波形的平滑度。

---

## 第一步：配置基本定时器 (TIM6)

1. **启用定时器**：在 STM32CubeMX 左侧菜单栏选择 **Timers -> TIM6**，勾选 **Activated** 启用定时器。
2. **配置频率参数**：在下方的 **Configuration -> Parameter Settings** 中设置：
   - **Prescaler (PSC - 16 bits value)**：假设系统时钟 ($f_{\text{TIM\_CLK}}$) 为 170MHz，设为 `170 - 1`，则分频后的定时器计数时钟为 1MHz。
   - **Counter Period (AutoReload Register - 16 bits value)**：假设希望 DAC 的触发频率为 10kHz，则设为 `100 - 1`。
   
   **计算公式：**
   $$f_{\text{trigger}} = \frac{f_{\text{TIM\_CLK}}}{(PSC + 1) \times (ARR + 1)}$$

3. **设置触发输出 (TRGO) 【关键步骤】**：
   - 找到 **Trigger Output (TRGO) Parameters**。
   - 将 **Trigger Event Selection** 设置为 **Update Event**。这表示每次定时器计数溢出（Update）时，都会向外发送一个触发信号。

![[Pasted image 20260531160421.png]]

---

## 第二步：配置DAC

1. **选择通道**：在左侧菜单栏选择 **Analog -> DAC1**（或其他需要使用的DAC）。
2. **配置输出引脚**：在 **OUT1 Configuration** 中：
   - 如果只需要输出到外部引脚，勾选 **Connected to external pin only**。
   - 如果内部还需要做比较器等联调，可选 **Connected to external pin and to on chip peripherals**。
   *(注：在我们自己的开发板上，由于有外置的运放电路，**Output Buffer** 项建议设为 **Disable**，否则可设为 **Enable** 以增强引脚驱动能力)*。
3. **配置参数 (Parameter Settings)**：
   - **Trigger**：选择 **Timer 6 Trigger Out event**（必须与上一步配置的定时器对应）。
   - **Wave generation mode**：设置为 **Disabled**（如果打算自己在代码里计算点位。如果是生成固定的三角波/噪声波，可以选择对应选项）。

![[屏幕截图 2026-05-31 161006.png]]

---

## 第三步：配置DMA

1. **添加通道**：在 DAC 的配置页面，切换到 **DMA Settings** 标签页。
2. **添加请求**：点击 **Add**，选择 `DAC1_CH1`。
3. **模式与数据宽度设置**：
   - **Mode**：设置为 **Circular**（循环模式，适合周期波形连续输出）。
   - **Data Width (Peripheral & Memory)**：均设置为 **Word**（根据实际数据对齐和寄存器宽度选择）。

![[屏幕截图 2026-05-31 163229.png]]

---

## 第四步：核心驱动代码实现

在初始化完成后，可以在业务代码中添加以下波形数据与启动代码：

```c
// 定义64点的正弦波/任意波形数据
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

void DAC_Wave_Start(void)
{
    // 1. 开启 DAC 的 DMA 传输  
    // 参数依次为：DAC句柄, 通道, 数据首地址(强转为uint32_t*), 数组长度, 数据对齐方式  
    HAL_DAC_Start_DMA(&hdac1, DAC_CHANNEL_1, (uint32_t*)wave_data_64, 64, DAC_ALIGN_12B_R);  
      
    // 2. 启动 TIM6 定时器 (用作DAC触发源)  
    // 注意：如果您的定时器句柄不是htim6，请替换为您实际使用的定时器句柄  
    HAL_TIM_Base_Start(&htim6);
}