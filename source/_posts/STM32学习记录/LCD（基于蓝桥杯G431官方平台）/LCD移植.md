
---
title: 蓝桥杯嵌入式：TFT-LCD模块快速配置
date: 2026-04-11 10:15:52
math: false
tags: [蓝桥杯, STM32, LCD, 嵌入式开发]
categories: 蓝桥杯
---

***

# 蓝桥杯嵌入式开发：TFT-LCD 模块快速配置

## 1. 工程准备与底层配置

在开始编写 LCD 驱动代码前，必须完成以下准备工作：

1. **导入官方底层驱动**：将官方资源包中提供的 `lcd.c`、`lcd.h` 以及字库文件 `fonts.h` 复制并添加到你的工程目录中（注意在 Keil/IDE 中配置好 Include Path 头文件路径）。
2. **CubeMX 引脚配置**：在 STM32CubeMX 中，根据开发板原理图配置好 LCD 所需的对应引脚配置（通常为 GPIO 推挽输出，速度设置为 High）。

![配置完成](image.png)
---

## 2. LCD 屏幕初始化

根据往年蓝桥杯赛题的常见要求，界面背景通常为黑色，字体为白色。初始化代码**必须放置在 GPIO 等底层外设初始化完成之后**，主循环开始之前。

**代码示例：**

```c
/* USER CODE BEGIN 2 */
// 1. 初始化 LCD 底层硬件
LCD_Init();

// 2. 清除屏幕并设置背景色为黑色
LCD_Clear(Black);
LCD_SetBackColor(Black);

// 3. 设置文本颜色为白色
LCD_SetTextColor(White);
/* USER CODE END 2 */
```

---

## 3. 静态字符串显示与编译警告修复

在完成初始化后，可以打印一行测试字符串以验证屏幕是否正常工作。

**基础打印指令：**
```c
LCD_DisplayStringLine(Line1, "   STM32G431RBT6    ");
```

**⚠️ 警告处理 (Warning Fix)：**
如果直接像上面那样传入字符串常量，编译器通常会报出类型不匹配的警告：
> `note: expected 'u8 *' {aka 'unsigned char *'} but argument is of type 'char *'`
> `182 | void LCD_DisplayStringLine(u8 Line, u8 *ptr);`

**解决方法：**
这是由于官方库函数要求的入参是 `uint8_t *` (即无符号字符指针)，而默认字符串是 `char *`。只需要在传入参数前加上强制类型转换即可完美消除警告：

```c
// 标准规范写法
LCD_DisplayStringLine(Line1, (uint8_t *)"   STM32G431RBT6    ");
```

---

## 4. 动态数据刷新与残影消除 (核心技巧)

在显示动态变化的变量（如 ADC 采集值、RTC 时间）时，如果新数据的位数比旧数据短（例如从 `100` 变成 `99`），屏幕上容易残留旧数据的最后一位（变成 `990`）。

**最佳实践方案：利用 C 语言格式化占位符补齐 20 位**（LCD 单行最多显示 20 个字符）。

```c
char LCD_DataTemp[30]; // 用于存放初次格式化的字符串
char LCD_DataLine1[30]; // 用于存放最终补齐空格的字符串

// 方法一：手动在 sprintf 中补齐末尾空格 (确保双引号内总计20个字符)
sprintf(LCD_DataTemp, "    Data1 = %d       ", Data1); 
LCD_DisplayStringLine(Line1, (uint8_t *)LCD_DataTemp);

// 方法二 (推荐)：利用格式化字符串自动补齐空格消影
sprintf(LCD_DataTemp, "    Data1 = %d", Data1);
// "%-20s" 表示左对齐，如果字符串长度不足20，则在右侧自动填充空格补齐
sprintf(LCD_DataLine1, "%-20s", LCD_DataTemp); 
LCD_DisplayStringLine(Line1, (uint8_t *)LCD_DataLine1);
```
*注：采用补齐空格的方式覆盖旧画面，可以有效避免频繁调用 `LCD_Clear()` 导致的屏幕剧烈闪烁。*

---

## 5. 多页面 (UI) 切换逻辑架构

在蓝桥杯考试中，经常会遇到“数据界面”、“设置界面”等多个页面的切换需求。使用 `switch-case` 状态机是最清晰、最不容易卡死主循环的写法。

**架构示例：**

```c
// 定义页面状态枚举或静态变量
static uint8_t LCD_Page = 1; 

// 在主循环 (while 1) 或定时器刷新任务中调用
switch(LCD_Page) 
{
    case 1:
        // 第一页：数据展示界面
        // UI_Page_Data_Show(); 
        break;
        
    case 2:
        // 第二页：参数设置界面
        // UI_Page_Setting_Show(); 
        break;
        
    // 依此类推...
    default:
        break;
}
```
**翻页控制逻辑：** 在按键处理函数中，当检测到特定的按键（如“切换按键”）按下时，改变 `LCD_Page` 的值，即可实现界面的平滑切换。每次切换页面时，建议触发一次局部清屏，以防不同页面的排版互相干扰。