---
title: "放弃 Keil5：基于 CLion + CMake + GCC 搭建 GD32F470 嵌入式开发环境"
date: 2026-04-14 23:40:00
math: false
categories: 
  - [嵌入式开发, GD32, 西门子杯]
tags: [GD32, 西门子杯, 环境配置, GCC, CMake, 嵌入式开发]
---


由于参加西门子杯比赛的需求，我不得不开启一段全新的 MCU 开发路程——GD32。之前很长一段时间就听说 GD32 和 STM32 高度相似，甚至在很多外设底层和代码上可以直接通用。

起初在配置开发环境的过程中，我发现绝大多数的教程使用的依然是 Keil5。但是，作为 CLion 的忠实用户，看着 Keil5 颇具年代感的界面，我决定挑战一下：用 CLion 搭建一套现代化的 GD32 开发环境。

既然它和 STM32 如此相似，我决定复用之前开发 STM32 时的经验，采用 **CLion + CMake + GCC 交叉编译工具链**，搭配 **ST-Link + OpenOCD** 的调试方案。

在查阅了官方的固件库后，我发现其原始架构有些散乱。为了后续开发的高效，我结合 STM32CubeMX 的代码生成习惯，自己重新定义了一套熟悉的工程架构。

---

## 一、 工程目录架构设计

我的自定义工程目录结构如下，如果你经常使用 CubeMX，一定会觉得非常眼熟：

```text
GD32F470_Project/
├── build/                 # 编译输出目录 (CMake 自动生成)
├── cmake/                 # CMake 工具链配置目录
│   ├── gcc-arm-none-eabi.cmake  # GCC 交叉编译工具链配置
│   └── starm-clang.cmake
├── Core/                  # 核心业务代码
│   ├── Inc/               # 存放 main.h, gd32f4xx_libopt.h (外设库裁剪宏) 等
│   └── Src/               # 存放 main.c, gd32f4xx_it.c (中断服务函数) 等
├── Drivers/               # GD32 官方固件库目录
│   ├── CMSIS/             # 包含 core_cm4.h 等 ARM 内核头文件及 GD32F4xx 寄存器定义
│   └── Library/           # GD32F4xx 标准外设库和 USB 库
├── stlink_gd32f470.cfg    # OpenOCD 调试配置文件
├── GD32F470VET6_FLASH.ld  # GCC 链接脚本 (极其重要)
├── startup_gd32f407_427.s # 启动文件 (注意需为 GCC 版本)
└── CMakeLists.txt         # 项目构建脚本
```

---

## 二、 核心配置文件详解

搭建这套环境的核心在于让 CMake 和 GCC 能够正确识别 GD32 的固件库、芯片内存分布以及启动逻辑。以下是各个核心文件的配置详情。

### 1. CMakeLists.txt 构建脚本

这个文件负责组织整个工程的编译过程，去除了对 STM32CubeMX 的依赖，并引入了 GD32 的标准库。

```cmake
cmake_minimum_required(VERSION 3.22)

# 1. 设置项目名称
set(CMAKE_PROJECT_NAME GD32F470_Project)
project(${CMAKE_PROJECT_NAME} C ASM)

# 启用 clangd 编译命令导出，方便代码补全和跳转
set(CMAKE_EXPORT_COMPILE_COMMANDS TRUE)
set(CMAKE_C_STANDARD 11)

# 2. 【关键】在此处提前定义目标
# 只有先执行了这一行，后续的 target_include_directories 才能找到挂载对象
add_executable(${CMAKE_PROJECT_NAME})

# 3. 宏定义 (GD32F470 核心)
target_compile_definitions(${CMAKE_PROJECT_NAME} PRIVATE
        GD32F470
        USE_STDPERIPH_DRIVER
)

# 4. 包含目录 (Include Paths)
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
        "Core/Inc"
        "Drivers/CMSIS"
        "Drivers/CMSIS/Include"             # 存放通用的 core_cm4.h
        "Drivers/CMSIS/GD/GD32F4xx/Include"
        "Drivers/Library/GD32F4xx_standard_peripheral/Include"
        "Drivers/Library/GD32F4xx_usb_library/driver/Include"
        "Drivers/Library/GD32F4xx_usb_library/device/Include"
        "Drivers/Library/Third_Party"
)

# 5. 源文件搜寻
file(GLOB_RECURSE CORE_SOURCES "Core/Src/*.c")
file(GLOB_RECURSE PERIPH_SOURCES "Drivers/Library/GD32F4xx_standard_peripheral/Source/*.c")

# 系统初始化与 GCC 版本的启动文件
set(STARTUP_SYSTEM_SOURCES
        "Drivers/CMSIS/GD/GD32F4xx/Source/system_gd32f4xx.c"
        "startup_gd32f407_427.s"
)

# 将搜寻到的源文件添加到目标
target_sources(${CMAKE_PROJECT_NAME} PRIVATE
        ${CORE_SOURCES}
        ${PERIPH_SOURCES}
        ${STARTUP_SYSTEM_SOURCES}
)

# 6. 链接设置
# 挂载我们自定义的 GD32F470 链接脚本
set(LINKER_SCRIPT "${CMAKE_CURRENT_SOURCE_DIR}/GD32F470VET6_FLASH.ld")

target_link_options(${CMAKE_PROJECT_NAME} PRIVATE
        -T${LINKER_SCRIPT}
        -Wl,-Map=${CMAKE_PROJECT_NAME}.map
        -Wl,--gc-sections
)

target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE
        m
        c
        nosys
)

# 7. 编译后操作：生成 Hex 和 Bin 烧录文件
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.hex
        COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.bin
)
```

### 2. OpenOCD 调试配置 (`stlink_gd32f470.cfg`)

为了让 CLion 能够通过 ST-Link 顺利连接并调试 GD32，我们需要手写一份 cfg 文件。需要特别注意 `CPUTAPID` 的设置。

```tcl
# 选择调试器接口: ST-Link
source [find interface/stlink.cfg]

# 选择传输协议: SWD
transport select hla_swd

# 设置芯片的 CPUTAPID
# GD32F470 的 Cortex-M4 内核 ID 通常与 STM32F4 相同 (0x2ba01477)
# 如果 OpenOCD 报错识别不到芯片，可以尝试将其设置为 0 (强制忽略 ID 检查)
set CPUTAPID 0x2ba01477

# 选择目标芯片配置文件
# GD32F470 的寄存器映射和调试逻辑完全兼容 STM32F4x，因此直接复用脚本
source [find target/stm32f4x.cfg]

# 设置下载速率 (可根据连接稳定性在 2000-8000 之间调整)
adapter speed 4000
```

### 3. GCC 链接脚本 (`GD32F470VET6_FLASH.ld`)

链接脚本决定了程序的代码和数据在芯片 Flash 和 RAM 中的存放位置。GD32F470VET6 拥有 **512KB Flash** 和 **256KB RAM**（其中包含 192KB 主 SRAM 和 64KB TCMSRAM）。

```ld
/* Entry Point */
ENTRY(Reset_Handler)

/* Specify the memory areas */
MEMORY
{
  /* GD32F470VET6: 512KB Flash, 256KB Total RAM */
  /* 主 RAM (SRAM0+SRAM1+SRAM2) */
  RAM (xrw)      : ORIGIN = 0x20000000, LENGTH = 192K
  /* TCMSRAM (对应 STM32 的 CCMRAM 区域) */
  CCMRAM (xrw)   : ORIGIN = 0x10000000, LENGTH = 64K
  FLASH (rx)     : ORIGIN = 0x08000000, LENGTH = 512K
}

/* 堆栈顶地址更新 */
_estack = ORIGIN(RAM) + LENGTH(RAM);
/* Generate a link error if heap and stack don't fit into RAM */
_Min_Heap_Size = 0x200;      /* required amount of heap  */
_Min_Stack_Size = 0x400;     /* required amount of stack */

/* Define output sections */
SECTIONS
{
  /* The startup code goes first into FLASH */
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector)) /* Startup code */
    . = ALIGN(4);
  } >FLASH

  /* The program code and other data goes into FLASH */
  .text :
  {
    . = ALIGN(4);
    *(.text)           /* .text sections (code) */
    *(.text*)          /* .text* sections (code) */
    *(.glue_7)         /* glue arm to thumb code */
    *(.glue_7t)        /* glue thumb to arm code */
    *(.eh_frame)

    KEEP (*(.init))
    KEEP (*(.fini))

    . = ALIGN(4);
    _etext = .;        /* define a global symbols at end of code */
  } >FLASH

  /* Constant data goes into FLASH */
  .rodata :
  {
    . = ALIGN(4);
    *(.rodata)         /* .rodata sections (constants, strings, etc.) */
    *(.rodata*)        /* .rodata* sections (constants, strings, etc.) */
    . = ALIGN(4);
  } >FLASH

  .ARM.extab (READONLY) : 
  {
    . = ALIGN(4);
    *(.ARM.extab* .gnu.linkonce.armextab.*)
    . = ALIGN(4);
  } >FLASH

  .ARM (READONLY) : 
  {
    . = ALIGN(4);
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
    . = ALIGN(4);
  } >FLASH

  .preinit_array (READONLY) : 
  {
    . = ALIGN(4);
    PROVIDE_HIDDEN (__preinit_array_start = .);
    KEEP (*(.preinit_array*))
    PROVIDE_HIDDEN (__preinit_array_end = .);
    . = ALIGN(4);
  } >FLASH

  .init_array (READONLY) : 
  {
    . = ALIGN(4);
    PROVIDE_HIDDEN (__init_array_start = .);
    KEEP (*(SORT(.init_array.*)))
    KEEP (*(.init_array*))
    PROVIDE_HIDDEN (__init_array_end = .);
    . = ALIGN(4);
  } >FLASH

  .fini_array (READONLY) : 
  {
    . = ALIGN(4);
    PROVIDE_HIDDEN (__fini_array_start = .);
    KEEP (*(SORT(.fini_array.*)))
    KEEP (*(.fini_array*))
    PROVIDE_HIDDEN (__fini_array_end = .);
    . = ALIGN(4);
  } >FLASH

  _siccmram = LOADADDR(.ccmram);

  .ccmram :
  {
    . = ALIGN(4);
    _sccmram = .;       /* create a global symbol at ccmram start */
    *(.ccmram)
    *(.ccmram*)
    . = ALIGN(4);
    _eccmram = .;       /* create a global symbol at ccmram end */
  } >CCMRAM AT> FLASH

  _sidata = LOADADDR(.data);

  .data :
  {
    . = ALIGN(4);
    _sdata = .;        /* create a global symbol at data start */
    *(.data)           /* .data sections */
    *(.data*)          /* .data* sections */
    *(.RamFunc)        /* .RamFunc sections */
    *(.RamFunc*)       /* .RamFunc* sections */
    . = ALIGN(4);
  } >RAM AT> FLASH

  .tdata : ALIGN(4)
  {
    *(.tdata .tdata.* .gnu.linkonce.td.*)
    . = ALIGN(4);
    _edata = .;        
    PROVIDE(__data_end = .);
    PROVIDE(__tdata_end = .);
  } >RAM AT> FLASH

  PROVIDE( __tdata_start = ADDR(.tdata) );
  PROVIDE( __tdata_size = __tdata_end - __tdata_start );
  PROVIDE( __data_start = ADDR(.data) );
  PROVIDE( __data_size = __data_end - __data_start );
  PROVIDE( __tdata_source = LOADADDR(.tdata) );
  PROVIDE( __tdata_source_end = LOADADDR(.tdata) + SIZEOF(.tdata) );
  PROVIDE( __tdata_source_size = __tdata_source_end - __tdata_source );
  PROVIDE( __data_source = LOADADDR(.data) );
  PROVIDE( __data_source_end = __tdata_source_end );
  PROVIDE( __data_source_size = __data_source_end - __data_source );

  .tbss (NOLOAD) : ALIGN(4)
  {
    _sbss = .;         
    __bss_start__ = _sbss;
    *(.tbss .tbss.*)
    . = ALIGN(4);
    PROVIDE( __tbss_end = . );
  } >RAM

  PROVIDE( __tbss_start = ADDR(.tbss) );
  PROVIDE( __tbss_size = __tbss_end - __tbss_start );
  PROVIDE( __tbss_offset = ADDR(.tbss) - ADDR(.tdata) );
  PROVIDE( __tls_base = __tdata_start );
  PROVIDE( __tls_end = __tbss_end );
  PROVIDE( __tls_size = __tls_end - __tls_base );
  PROVIDE( __tls_align = MAX(ALIGNOF(.tdata), ALIGNOF(.tbss)) );
  PROVIDE( __tls_size_align = (__tls_size + __tls_align - 1) & ~(__tls_align - 1) );
  PROVIDE( __arm32_tls_tcb_offset = MAX(8, __tls_align) );
  PROVIDE( __arm64_tls_tcb_offset = MAX(16, __tls_align) );

  .bss (NOLOAD) : ALIGN(4)
  {
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;         
    __bss_end__ = _ebss;
    PROVIDE( __bss_end = .);
  } >RAM
  
  PROVIDE( __non_tls_bss_start = ADDR(.bss) );
  PROVIDE( __bss_start = __tbss_start );
  PROVIDE( __bss_size = __bss_end - __bss_start );

  ._user_heap_stack (NOLOAD) :
  {
    . = ALIGN(8);
    PROVIDE ( end = . );
    PROVIDE ( _end = . );
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
  } >RAM

  /DISCARD/ :
  {
    libc.a:* ( * )
    libm.a:* ( * )
    libgcc.a:* ( * )
  }
}
```

---

## 三、启动文件与链接脚本的对齐

在这里必须着重记录一个巨大的坑。

GD32 官方固件库提供的 GCC 版 `startup_gd32f407_427.s` 启动文件，其默认的宏命名与上文配置的 `.ld` 链接脚本存在差异。如果不统一，会导致编译出的固件丢失中断向量表，单片机一上电 CPU 指针就会跑飞（卡死在无数个 `movs r0, r0` 空指令中）。

**我们需要在 `.s` 启动文件中修改两个地方：**

1. 向量表存放的**段名**没对上（链接器找不到位置）。
2. **栈顶指针变量名**没对上（导致上电后 MSP 错误）。

在 `startup_gd32f407_427.s` 文件中找到定义中断向量表的位置进行修改：

**❌ 官方默认的错误代码：**
```assembly
.section  .vectors,"a",%progbits       /* ❌ 错误 1：段名不对，链接脚本中叫 .isr_vector */
   .global __gVectors

__gVectors:
                    .word _sp             /* ❌ 错误 2：栈顶指针不对，链接脚本中叫 _estack */
                    .word Reset_Handler   /* Reset Handler */
```

**✅ 修正后的正确代码：**
```assembly
.section  .isr_vector,"a",%progbits    /* ✅ 修正：改为与 ld 脚本一致的 .isr_vector */
   .global __gVectors

__gVectors:
                    .word _estack         /* ✅ 修正：改为与 ld 脚本一致的 _estack */
                    .word Reset_Handler   /* Reset Handler */
```

修改完毕后，清理 CMake 缓存并重新编译下载，代码终于稳稳地跳进了 `main()` 函数，LED 成功闪烁了起来！

自此，GD32F470 的现代版 CLion 开发环境全部配置完成！
