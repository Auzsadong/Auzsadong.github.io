---
title: STM32开发备忘录：EEPROM (AT24C02) 软件 I2C 读写时序与驱动
date: 2026-04-11 22:45:00
math: false
categories: 
  - [嵌入式开发, STM32]
  - [蓝桥杯]
tags: [I2C, EEPROM, STM32, 蓝桥杯, 笔记, 嵌入式开发]
---

# STM32开发备忘录：EEPROM 软件 I2C 底层驱动与时序解析

在蓝桥杯和常规嵌入式开发中，EEPROM（通常为 AT24C02）是用于掉电保存关键数据的核心外设。由于 IO 口分配的原因，蓝桥杯平台的 EEPROM 接口被安放在了 **PB8 和 PB6** 引脚，为了稳定性和避免引脚冲突，我们直接使用官方提供的**软件 I2C 驱动文件**（`i2c.c` 和 `i2c.h`）进行底层操作。

本文将整理并封装最常用的单字节读写函数，并深度解析 I2C 的时序规范。

---

## 1. 核心器件地址定义

AT24C02 的 7 位器件地址通常为 `1010000` (0x50)。加上最低位的读写控制位（R/W）后，组合成 8 位地址：
* **写地址**：`0xA0` (末位为 0，表示 Write)
* **读地址**：`0xA1` (末位为 1，表示 Read)

```c
#define EEPROM_ADDR_WRITE 0xA0
#define EEPROM_ADDR_READ  0xA1
```

---

## 2. 核心驱动代码封装

在工程中包含官方提供的 I2C 驱动库后，我们可以基于其基础函数封装我们所需的 `WriteByte` 和 `ReadByte` 函数。

### 2.1 单字节写入函数

```c
// 向 EEPROM 指定地址写入一个字节的数据
void EEPROM_WriteByte(uint8_t addr, uint8_t data)
{
    I2CStart();                  // 1. 发送起始信号 (Start)
    I2CSendByte(EEPROM_ADDR_WRITE); // 2. 发送器件写地址 (0xA0)
    I2CWaitAck();                // 3. 等待器件应答 (ACK)

    I2CSendByte(addr);           // 4. 发送要写入的内部字节地址
    I2CWaitAck();                // 5. 等待器件应答

    I2CSendByte(data);           // 6. 发送要写入的实际数据
    I2CWaitAck();                // 7. 等待器件应答

    I2CStop();                   // 8. 发送停止信号 (Stop)

    // ⚠️ 极其重要：AT24C02 内部写周期需要约 5ms 的时间
    // 连续写入时如果不加此延时，会导致下一次写入失败或数据丢失
    HAL_Delay(5); 
}
```

### 2.2 单字节读取函数

读取操作比写入稍复杂，需要使用**伪写（Dummy Write）**技术来改变 EEPROM 的内部地址指针。

```c
// 从 EEPROM 指定地址读取一个字节的数据
uint8_t EEPROM_ReadByte(uint8_t addr)
{
    uint8_t data = 0;

    // ==============================================================
    // 第一步：伪写操作 (Dummy Write) —— 目的仅为设置内部地址指针
    // ==============================================================
    I2CStart();
    I2CSendByte(EEPROM_ADDR_WRITE); // 发送器件写地址 (0xA0)
    I2CWaitAck();
    I2CSendByte(addr);           // 发送要读取的内部地址指针
    I2CWaitAck();

    // ==============================================================
    // 第二步：改变数据流向，真正开始读取数据
    // ==============================================================
    I2CStart();                  // 重新发送起始信号 (Restart，非常关键)
    I2CSendByte(EEPROM_ADDR_READ);  // 发送器件读地址 (0xA1)
    I2CWaitAck();

    data = I2CReceiveByte();     // 接收一个字节的数据
    I2CSendNotAck();             // 发送非应答信号 (NACK)，告诉主机读完了，不要再发数据了

    I2CStop();                   // 发送停止信号 (Stop)

    return data;
}
```

---

## 3. 时序逻辑总结

在使用软件 I2C 操作 EEPROM 时，必须牢记以下三个时序法则：

1. **应答机制**：主机（单片机）每次发送完 1 个字节（无论是地址还是数据）后，都**必须**调用 `I2CWaitAck()` 等待从机（EEPROM）拉低 SDA 产生应答。
2. **读取时序的转换**：在读数据时，必须先按“写时序”发送一遍：`写地址 (0xA0) -> 目标内部地址`。接着，**务必重新产生一次 I2C 起始信号 (Restart)**，然后发送 `读地址 (0xA1)`，才能进入正常的接收模式。
3. **NACK 的作用**：在读取完所需的数据（本例中为 1 个字节）后，主机需要向从机发送一个非应答信号 `I2CSendNotAck()`，这相当于告诉从机：“我的数据收够了，你可以释放总线了”，随后才能发送 Stop 信号。

---

## 4. 业务逻辑初始化与调用实例

在 `main.c` 中使用时，千万别忘了先初始化软件 I2C 的 GPIO 引脚。

```c
uint8_t test_write = 0x55;
uint8_t test_read = 0x00;

/* USER CODE BEGIN 2 */
// 1. 初始化软件 I2C 引脚
I2CInit(); 

// 2. 将数据 0x55 写入 EEPROM 内部的 0x10 地址
EEPROM_WriteByte(0x10, test_write);

// 3. 从 EEPROM 的 0x10 地址将数据读回
test_read = EEPROM_ReadByte(0x10);
/* USER CODE END 2 */
```