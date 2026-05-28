# Zephyr 入门开发指南

> **本文不重复《Zephyr教程》中的环境搭建和基础语法。**
> **本文关注的是：当你要写一个模块时，该怎么思考、该去查什么、去哪儿查。**
> 适合在已有一定 Zephyr 基础（至少跑通过一个 sample）后阅读。

---

## 目录

- [第一章：Zephyr 开发的底层逻辑](#第一章zephyr-开发的底层逻辑)
- [第二章：设备树——属性名到底从哪来？](#第二章设备树属性名到底从哪来)
- [第三章：YAML Binding——怎么写？参考什么？](#第三章yaml-binding怎么写参考什么)
- [第四章：prj.conf——参数名从哪来？](#第四章prjconf参数名从哪来)
- [第五章：应用开发——我怎么知道该调哪个 API？](#第五章应用开发我怎么知道该调哪个-api)
- [第六章：实战案例——从零做一个外设驱动](#第六章实战案例从零做一个外设驱动)
- [附录：文件索引与速查](#附录文件索引与速查)

---

# 第一章：Zephyr 开发的底层逻辑

## 1.1 三驾马车

Zephyr 应用由三部分共同描述，**缺一不可**：

| 组成部分 | 文件 | 作用 | 类比 |
|---------|------|------|------|
| **Device Tree** | `.dts` / `.dtsi` / `.overlay` | 描述**硬件有什么** | 物料清单 |
| **Kconfig** | `prj.conf` / `Kconfig` / `Kconfig.defconfig` | 配置**软件开什么功能** | 菜单选择 |
| **C 代码** | `.c` / `.h` | 写**业务逻辑** | 操作手册 |

**关键原则：**

```
硬件相关的信息            → 放设备树（引脚号、外设地址、中断号）
软件可选的特性            → 放 Kconfig（要不要 shell、日志级别、栈大小）
硬件在板子上有没有        → 设备树（status = "okay" 或 "disabled"）
这个功能模块要不要编译    → Kconfig（CONFIG_xxx=y）
```

> 💡 **一个简易判断标准：**
> 如果换个板子就要改 → 那是硬件描述，放**设备树**。
> 如果同样的硬件只是配置不同 → 那是软件配置，放 **Kconfig**。

## 1.2 构建流程全景

理解文件是怎么串起来的，对排查问题特别重要：

```
你的应用目录/
├── src/main.c                ─┐
├── boards/myboard.overlay    ─┤
├── dts/bindings/xxx.yaml     ─┤
└── prj.conf                  ─┤
                               │
   west build                   │
       ↓                       ↓
   预处理阶段  ─────────────────┘
       │
       ├── DTS 文件被合并（含 overlay）
       ├── C 代码中的 DT_xxx 宏被展开成
       │    build/<...>/devicetree_generated.h 中的具体值
       └── Kconfig 被合并、求值 → 得到 .config
       ↓
   编译链接
       ↓
   生成 .elf / .hex / .bin
```

**查看"合并后"的设备树（调试神器）：**

```bash
# 编译后查看最终合并的设备树
cat build/zephyr/zephyr.dts          # 完整的设备树
cat build/zephyr/.config             # 最终 Kconfig 值

# 查看设备树 API 宏展开后的值
cat build/zephyr/include/generated/devicetree_generated.h
# 或者旧版本中叫 devicetree_unfixed.h
```

## 1.3 应用目录结构约定

Zephyr 应用的标准目录结构：

```
my_app/
├── src/
│   └── main.c
├── prj.conf              # 项目配置文件
├── boards/               # （可选）板级覆盖
│   └── myboard.overlay   # 设备树覆盖文件
├── CMakeLists.txt        # 构建定义
└── dts/                  # （可选）自定义 binding
    └── bindings/
        └── vendor,device.yaml
```

---

# 第二章：设备树——属性名到底从哪来？

这是新手最困惑的问题：我在设备树里写 `reg = <0x40005400 0x400>`，凭什么？`gpios` 和 `gpio` 有什么区别？

## 2.1 思考路径：找属性的完整决策树

```
你要知道某个设备树节点可以写什么属性？
│
├─ Step 1：确定 compatible 字符串
│  如 "bosch,bme280"、"gpio-leds"
│
├─ Step 2：去 dts/bindings/ 找对应的 yaml binding
│  find dts/bindings -name "*.yaml" | xargs grep -l "bosch,bme280"
│  或按类别浏览：ls dts/bindings/sensor/
│
├─ Step 3：打开 yaml，看 properties 和 child-binding
│  ┌ 列在 properties: 下面的 → 直接写（注意 required/optional）
│  ├ 有 include: xxx.yaml    → 看那个文件里又定义了什么属性
│  └ 属性类型决定了写法     → 参考 第三章 3.3 节
│
├─ Step 4：如果 yaml 没写，但你知道必须有
│  ┌ 检查是不是设备树规范属性（下表一）
│  ├ 检查是不是 Zephyr 扩展属性（下表二）
│  └ 检查驱动代码里有没有 DT_PROP/MACRO 解析它
│
└─ Step 5：还找不到？进驱动源码 grep
   grep -r "DT_PROP\|DT_INST_PROP" zephyr/drivers/<类别>/ --include="*.c"
```

## 2.2 属性的四大来源

### 来源一：Binding 文件规定的属性（最核心）

每个 `compatible` 字符串对应一个 YAML binding 文件，**这是优先查阅的来源**。

```
dts/bindings/<类别>/<设备名>.yaml
```

**怎么找 binding：**

```bash
# 方法 1：grep compatible 字符串
grep -r "myvendor,mydevice" dts/bindings/

# 方法 2：按类别浏览
ls dts/bindings/sensor/
ls dts/bindings/gpio/

# 方法 3：如果知道 compatible 但不确定所在类别
find dts/bindings -name "*.yaml" -exec grep -l "myvendor" {} \;
```

### 来源二：设备树规范/内核标准属性

这些**不是由某个驱动定义的**，而是设备树规范的一部分，所有设备树都遵循：

| 属性名 | 用途 | 说明 |
|--------|------|------|
| `compatible` | 匹配驱动 | 格式 `"vendor,device"` |
| `reg` | 地址/长度 | 配合 `#address-cells` / `#size-cells` |
| `status` | 启用/禁用 | `"okay"` / `"disabled"` |
| `interrupts` | 中断号 | 配合 `interrupt-parent` |
| `label` | 设备名（已弃用） | 新代码用设备树 API，不再用 label |
| `gpios` | GPIO 引脚 | phandle-array 格式 |
| `pinctrl-0` | 引脚复用 | 指向 pinctrl 节点 |
| `pinctrl-names` | 引脚复用名 | 如 `"default"`、`"sleep"` |

### 来源三：驱动自定义属性

有些驱动定义自己的专属属性，定义在**对应驱动的 binding yaml** 里：

```yaml
# dts/bindings/sensor/vendor,my-sensor.yaml
properties:
  int-gpios:                # ← 这是驱动专属的
    type: phandle-array
    required: true
    description: "Interrupt GPIO pin"

  measurement-mode:
    type: int
    required: false
    enum: [0, 1, 2]        # ← 可选属性，有限定值
    default: 0
```

**等价的设备树写法：**

```devicetree
my_sensor@48 {
    compatible = "vendor,my-sensor";
    reg = <0x48>;
    int-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;
    measurement-mode = <2>;
};
```

### 来源四：Zephyr 框架规定的扩展属性（`zephyr,` 前缀）

这些属性是 Zephyr 框架自身约定的，在 `chosen` 节点或其他框架节点中使用：

| 属性 | 说明 | 参考位置 |
|------|------|---------|
| `zephyr,console` | 指定控制台串口 | DT_CHOSEN(zephyr_console) |
| `zephyr,shell-uart` | 指定 shell 串口 | DT_CHOSEN(zephyr_shell_uart) |
| `zephyr,code` | 按键映射的输入码 | gpio-keys binding |
| `zephyr,flash` | 指定 flash 控制器 | chosen 节点 |
| `zephyr,sram` | 指定 SRAM 区域 | chosen 节点 |
| `zephyr,bt-hci` | 蓝牙 HCI 串口 | chosen 节点 |

## 2.3 实战：查找属性完整演示

### 场景：为一个 I2C 传感器 BME280 写设备树节点

**Step 1：知道是 I2C 传感器 → 找 I2C 类的 binding**

```bash
ls zephyr/dts/bindings/sensor/ | grep bme
# 输出：bosch,bme280.yaml
```

**Step 2：打开 yaml 看属性**

```yaml
# dts/bindings/sensor/bosch,bme280.yaml
compatible: "bosch,bme280"
include: [i2c-device.yaml]     # ← 继承了 I2C 设备规范

properties:
  int-gpios:                   # ← 只有这个驱动特有的
    type: phandle-array
    required: false
```

**Step 3：追踪 include 链**

`i2c-device.yaml` 在 `dts/bindings/i2c/i2c-device.yaml`：

```yaml
# 这个文件规定了所有 I2C 设备都需要的属性
properties:
  reg:
    required: true
    type: int            # I2C 地址，如 0x76
```

**Step 4：得出最终设备树写法**

```devicetree
&i2c1 {
    status = "okay";
    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;                  # 来自 i2c-device.yaml
        int-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;  # 来自 bme280.yaml（可选）
    };
};
```

**Step 5：验证**

```bash
# 编译后查看生成的宏
grep "BME280" build/zephyr/include/generated/devicetree_generated.h
```

## 2.4 Overlay 文件机制

当你不想修改原版 dts 文件时（比如用了官方开发板），用 overlay 来覆盖或添加节点。

**Overlay 文件的搜索路径：**

```
应用目录/boards/<BOARD_NAME>.overlay   ← 最常用
应用目录/<BOARD_NAME>.overlay
boards/<BOARD_NAME>.overlay
```

或者通过 `west build` 参数指定：

```bash
west build -b nucleo_f411re . -DOVERLAY_CONFIG=my.overlay
```

**Overlay 常见写法：**

```devicetree
// 1. 启用已在 soc dtsi 中定义但默认 disabled 的外设
&i2c1 {
    status = "okay";
};

// 2. 在已有总线下添加新设备
&i2c1 {
    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
    };
};

// 3. 修改现有节点属性
&uart0 {
    current-speed = <115200>;
};

// 4. 添加全新节点
/ {
    my_thing: my-thing {
        compatible = "myvendor,my-thing";
        some-property;
    };
};
```

## 2.5 特殊节点类型详解

### GPIO 控制器节点

GPIO 控制器在 SoC 的 `.dtsi` 中已定义，一般不需要手动写，只需要**引用**：

```devicetree
my-gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;
```

- `&gpio0` → 指向 GPIO 控制器（在 .dtsi 里定义）
- `13` → 引脚号
- `GPIO_ACTIVE_LOW` → 电平极性，定义在 `dt-bindings/gpio/gpio.h`

**如何找到 GPIO 控制器名：**

```bash
grep -r "gpio@" dts/arm/st/f4/stm32f405.dtsi
# 输出类似：
# gpio_a: gpio@40020000 { ... };
# gpio_b: gpio@40020400 { ... };
# ...
```

### Pinctrl 节点

引脚复用配置，通常在板级 pinctrl dtsi 文件中：

```devicetree
&pinctrl {
    uart0_default: uart0_default {
        group1 {
            psels = <STM32_PSEL_UART_TX(PB6)>;
        };
    };
};
```

`STM32_PSEL_xxx` 宏定义位置：

```
zephyr/include/zephyr/dt-bindings/pinctrl/<芯片系列>.h
```

例如 STM32 系列是 `stm32f4-pinctrl.h` 或 `stm32-common-pinctrl.h`。

## 2.6 设备树调试三板斧

```bash
# 1. 看合并后的完整设备树
cat build/zephyr/zephyr.dts

# 2. 看生成的头文件（宏展开后的值）
cat build/zephyr/include/generated/devicetree_generated.h

# 3. 查看 DT API 测试（快速验证某个宏能展开成什么）
# 写个临时测试：
#include <zephyr/devicetree.h>
DT_ALIAS(led0)       # 在脑子里展开，或用编译后的头文件确认
```

## 2.7 STM32 设备树注意事项

如果你是 STM32 用户，有两点特别容易踩坑：

### 时钟配置（RCC）

STM32 的时钟配置不在 Kconfig 里，而在**设备树**中。这是很多从 STM32CubeMX/HAL 转过来的人不习惯的地方。

时钟配置在 SoC 级 .dtsi 中定义，板级 .dts 或 overlay 中修改。以 STM32F4 为例：

```devicetree
/* 在板级 .dts 或 overlay 中 */
&clk_hse {
    /* 外部高速晶振频率（Hz）*/
    hse-clk = <8000000>;
    hse-ready-hsi = <1>;          /* 等待 HSE 稳定期间保持 HSI 运行 */
    status = "okay";
};

&pll {
    /* PLL 配置：PLL_M, PLL_N, PLL_P, PLL_Q */
    pll-mul = <168>;              /* PLL_N = 168 */
    pll-div = <4>;                /* PLL_M = 4 */
    pll-p = <2>;                  /* PLL_P = 2（得到 84MHz PLL/P） */
    pll-q = <7>;                  /* PLL_Q = 7（用于 USB/SDIO） */
    status = "okay";
};

&rcc {
    clocks = <&pll>;              /* 选择 PLL 作为系统时钟源 */
    clock-frequency = <DT_HZ_168000000>;
};
```

**如何查看你的芯片需要哪些时钟节点：**

```bash
# 搜索芯片的 soc dtsi 中以 clk_ 开头的节点
grep "clk_" zephyr/dts/arm/st/f4/stm32f405.dtsi | head -20
# 会看到 clk_hse, clk_lse, pll 等节点
```

### Flash 分区（MCUboot 或 OTA 时需要）

```devicetree
/ {
    chosen {
        /* 必须指定 flash 分区 */
        zephyr,code-partition = &slot0_partition;
    };
};

&flash0 {
    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        boot_partition: partition@0 {
            label = "mcuboot";
            reg = <0x00000000 0x00010000>;    /* 64KB bootloader */
        };
        slot0_partition: partition@10000 {
            label = "image-0";
            reg = <0x00010000 0x00060000>;    /* 384KB app slot 0 */
        };
        slot1_partition: partition@70000 {
            label = "image-1";
            reg = <0x00070000 0x00060000>;    /* 384KB app slot 1 */
        };
    };
};
```

---

# 第三章：YAML Binding——怎么写？参考什么？

## 3.1 什么时候需要写 Binding

| 场景 | 需要写 Binding？ | 怎么办 |
|------|-----------------|--------|
| 用现成的官方外设（UART、I2C、GPIO） | ❌ 不需要 | 直接用已有的 binding |
| 用现成的官方传感器芯片 | ❌ 一般不用 | Zephyr 自带，去 samples/ 找示例 |
| 把自己的传感器驱动加进 Zephyr | ✅ 需要写 | 按本文指引 |
| 给板子加自定义硬件（如 FPGA 映射） | ✅ 需要写 | 写简单的 binding |
| 临时调试，加一个测试节点 | ⚠️ 看情况 | 可以写 .overlay 但不写 binding，只要不用 DT_DRV_COMPAT |

## 3.2 Binding 文件结构（完整模板）

```yaml
# dts/bindings/<类别>/<厂商>,<设备名>.yaml
# 文件名必须和 compatible 一致

# --- 1. 匹配 compatible ---
compatible: "myvendor,mydevice"

# --- 2. 继承 ---
include: [i2c-device.yaml]
# 或 include 多个：
include:
  - i2c-device.yaml
  - pinctrl-device.yaml

# --- 3. 描述 ---
description: |
  My custom device. This is a multi-line
  description of what this device does.

# --- 4. 属性定义 ---
properties:
  my-gpios:
    type: phandle-array
    required: true
    description: "My custom GPIO pin"

  config-value:
    type: int
    required: false
    default: 100
    enum:
      - 100
      - 200
      - 500

# --- 5. 子节点定义（可选） ---
# 当父节点可以有多个子节点时使用
child-binding:
  description: Child node
  properties:
    reg:
      type: int
      required: true
```

## 3.3 属性类型与写法对照

| type | 含义 | 示例 |
|------|------|------|
| `int` | 单个整数 | `<0x76>` |
| `array` | 整数数组 | `<0x100 0x200>` |
| `string` | 字符串 | `"my-device"` |
| `string-array` | 字符串数组 | `["a", "b"]` |
| `boolean` | 布尔（存在即为 true） | `wakeup-source;` |
| `phandle` | 指向另一个节点 | `<&gpio0>` |
| `phandle-array` | phandle + 参数 | `<&gpio0 13 GPIO_ACTIVE_LOW>` |
| `uint8-array` | 字节数组（hex） | `[12 34 56]` |

## 3.4 常用的 include 基类

| 基类文件 | 包含内容 | 使用场景 |
|---------|---------|---------|
| `i2c-device.yaml` | `reg`（I2C 地址）、`#address-cells` | I2C 外设 |
| `spi-device.yaml` | `reg`（CS 号） | SPI 外设 |
| `gpio-interface.yaml` | `gpios` | GPIO 接口设备 |
| `pinctrl-device.yaml` | `pinctrl-0`、`pinctrl-names` | 引脚复用控制的设备 |
| `on-bus.yaml` | 基本总线属性 | 在总线上的设备 |

**继承链示例：**

```yaml
# bosch,bme280.yaml
compatible: "bosch,bme280"
include: [i2c-device.yaml]    # → 得到 reg
properties:
  int-gpios: ...               # 自定义属性
```

## 3.5 最佳学习方法：参考同类

| 你想写什么 | 参考哪个 binding |
|-----------|----------------|
| I2C 传感器 | `dts/bindings/sensor/bosch,bme280.yaml` |
| SPI 设备 | `dts/bindings/sensor/adi,adxl362.yaml` |
| GPIO 输出（LED） | `dts/bindings/led/gpio-leds.yaml` |
| GPIO 输入（按键） | `dts/bindings/gpio/gpio-keys.yaml` |
| Flash 分区 | `dts/bindings/mtd/partitions.yaml` |
| PWM 输出 | `dts/bindings/pwm/pwm-leds.yaml` |
| 最简模板 | `dts/bindings/gpio/gpio-leds.yaml` |

## 3.6 Binding 验证方法

```bash
# 方法 1：编译看是否报 binding 错误
west build -b your_board .

# 方法 2：查看生成的头文件中是否包含你的节点
cat build/zephyr/include/generated/devicetree_generated.h | grep "MYDEVICE"
# 或旧的路径：
cat build/zephyr/include/generated/devicetree_unfixed.h | grep "MYDEVICE"

# 方法 3：查看最终合并的设备树
cat build/zephyr/zephyr.dts | grep -A 10 "mydevice"
```

---

# 第四章：prj.conf——参数名从哪来？

## 4.1 Kconfig 参数定义的核心逻辑

每个 `CONFIG_XXX` 都在 Zephyr 源码树**某个地方被定义了**。定义它的位置决定了它的用途。

**Kconfig 文件分布（按查找优先级排序）：**

```
zephyr/Kconfig                       → 全局、内核基础（线程数、栈大小...）
zephyr/drivers/<类别>/Kconfig        → 驱动使能（GPIO、I2C、SPI...）
zephyr/subsys/<子系统>/Kconfig       → 子系统（Shell、Log、文件系统...）
zephyr/modules/<模块>/Kconfig        → 外部模块（FatFS、LVGL...）
boards/<厂商>/<板名>/Kconfig.defconfig → 板级默认值
arch/<架构>/Kconfig                  → 架构相关（ARM、RISC-V...）
```

## 4.2 五种查找方法（逐个掌握）

### 方法 1：menuconfig（首选）

```bash
west build -t menuconfig   # 终端界面（最常用、最稳定）
# 或
west build -t guiconfig    # 图形界面（更友好，但有的环境不支持）
```

**menuconfig 使用技巧：**

1. 按 `/` 键打开搜索
2. 输入关键词（如 `I2C`、`LOG`）
3. 搜索结果中会显示该选项的完整符号名和路径
4. 按 `?` 看帮助——里面写明了依赖关系、默认值
5. 按 `1/2/3` 跳转到对应的菜单位置

### 方法 2：grep 搜索

```bash
# 注意：搜的是 "config XXX" 不是 "CONFIG_XXX"
grep -r "config I2C_STM32" zephyr/ --include="Kconfig*"

# 如果找不到，可能是嵌套在 if/choice 块中
grep -r "I2C_STM32" zephyr/ --include="Kconfig*" -A5
```

### 方法 3：看硬件板子的 defconfig

同系列 SoC 的官方板子配置是最好的参考：

```bash
cat zephyr/boards/arm/nucleo_f401re/nucleo_f401re_defconfig
# 里面列出所有此板子默认启用的 CONFIG
```

### 方法 4：看官方 sample 的 prj.conf

```bash
cat zephyr/samples/drivers/gpio/prj.conf
cat zephyr/samples/sensor/bme280/prj.conf
```

### 方法 5：查看编译结果

```bash
# 实际生效的配置（比 prj.conf 多很多，因为包含了默认值和依赖推导）
cat build/zephyr/.config | grep -E "(I2C|UART|SHELL)" | head -20
```

## 4.3 完整追踪示例：一个参数的生命周期

### 案例：你想知道 `CONFIG_I2C_STM32=y` 是从哪来的。

```bash
# Step 1：先搜定义
grep -r "config I2C_STM32" zephyr/ --include="Kconfig*"
# 结果在：zephyr/drivers/i2c/Kconfig.stm32
# 或者：zephyr/drivers/i2c/Kconfig

# Step 2：打开看
cat zephyr/drivers/i2c/Kconfig.stm32 | head -30
# 你会看到类似：
# config I2C_STM32
#     bool "STM32 I2C driver"
#     depends on I2C && $(dt_compat_enabled,$(DT_COMPAT_ST_STM32_I2C))
#     help
#       Enable STM32 I2C driver

# Step 3：理解依赖
# "depends on I2C" → 必须先有 CONFIG_I2C=y
# 设备树检查 → 设备树中有 compatible = "st,stm32-i2c" 的节点

# Step 4：你在 prj.conf 里只需要写
CONFIG_I2C=y
CONFIG_I2C_STM32=y
```

### 案例：想了解 CONFIG_LOG 都有哪些子选项

```bash
# Step 1：menuconfig 搜索
west build -t menuconfig
# 按 / → 输入 log → 看到所有相关选项

# Step 2：或者直接命令行搜
grep -r "config LOG" zephyr/subsys/logging/ --include="Kconfig*"
```

常用的日志相关选项：

```
CONFIG_LOG=y              # 启用日志子系统
CONFIG_LOG_DEFAULT_LEVEL=3  # 级别：4=DBG, 3=INF, 2=WRN, 1=ERR, 0=OFF
CONFIG_LOG_BACKEND_UART=y # 串口日志后端
```

## 4.4 常用参数分类速查

### 驱动使能（先开框架，再开具体驱动）

```
CONFIG_GPIO=y          # GPIO 子系统框架
CONFIG_I2C=y           # I2C 子系统框架
CONFIG_SPI=y           # SPI 子系统框架
CONFIG_ADC=y           # ADC 子系统框架
CONFIG_PWM=y           # PWM 子系统框架
CONFIG_SENSOR=y        # 传感器子系统框架
CONFIG_SERIAL=y        # 串口子系统框架
```

### 具体驱动（框架之上的具体芯片驱动）

```
CONFIG_GPIO_STM32=y             # STM32 GPIO 驱动
CONFIG_I2C_STM32=y              # STM32 I2C 驱动
CONFIG_SENSOR_BME280=y          # BME280 传感器驱动
CONFIG_ADC_STM32=y              # STM32 ADC 驱动
```

> 🎯 **STM32 用户小提示：** board 的 `*_defconfig` 一般已经配好了 SoC 型号（`CONFIG_SOC_STM32F411XE` 等）和该系列对应的基本驱动。你只需要补充自己新增的外设即可，不需要重复配已有的。
> `CONFIG_SOC_SERIES_STM32F4X=y` 这类通常由 board 的 `Kconfig.defconfig` 设置，prj.conf 中不用写。

### 系统功能

```
CONFIG_SHELL=y                  # Shell 命令行
CONFIG_LOG=y                    # 日志系统
CONFIG_ASSERT=y                 # 断言（开发时开）
CONFIG_PRINTK=y                 # printk 输出（通常默认 y）
CONFIG_UART_CONSOLE=y           # UART 控制台
CONFIG_CONSOLE=y                # 控制台子系统
```

### 线程/内存

```
CONFIG_MAIN_STACK_SIZE=4096     # main 线程栈大小
CONFIG_HEAP_MEM_POOL_SIZE=4096  # 堆大小
CONFIG_NUM_PREEMPT_PRIORITIES=5 # 抢占优先级级数
CONFIG_NUM_COOP_PRIORITIES=5    # 协作优先级级数
```

### 调试

```
CONFIG_DEBUG=y                  # 调试模式
CONFIG_LOG_DEFAULT_LEVEL=4      # 日志级别：4=DBG, 3=INF
CONFIG_SHELL_BACKEND_SERIAL=y   # Shell 串口后端
CONFIG_DEVICE_SHELL=y           # 设备 shell 命令
```

---

# 第五章：应用开发——我怎么知道该调哪个 API？

## 5.1 核心思维：先选子系统，再找 API

**不要直接去翻头文件。先问自己：我要操作什么类型的硬件或功能？**

```
我要控制 LED
   └─ 属于 GPIO 输出
       └─ 子系统：GPIO → #include <zephyr/drivers/gpio.h>
           └─ API：gpio_pin_configure(), gpio_pin_set(), gpio_pin_toggle()

我要读温度传感器
   └─ 属于传感器
       └─ 子系统：Sensor → #include <zephyr/drivers/sensor.h>
           └─ API：sensor_sample_fetch(), sensor_channel_get()

我要在串口上打印信息 + 接收输入
   └─ 两个方向：
       ├─ 简单打印 → printk() / console 子系统
       └─ 交互式命令 → Shell 子系统 → CONFIG_SHELL=y
```

## 5.2 子系统 × 头文件 × API 查询表

| 你要做什么 | 包含的头文件 | 常用 API |
|-----------|------------|---------|
| GPIO 输入/输出 | `zephyr/drivers/gpio.h` | `gpio_pin_configure()`, `gpio_pin_set()`, `gpio_pin_get()`, `gpio_pin_interrupt_configure()` |
| I2C 读写 | `zephyr/drivers/i2c.h` | `i2c_write_read()`, `i2c_burst_read()`, `i2c_transfer()` |
| SPI 读写 | `zephyr/drivers/spi.h` | `spi_transceive()`, `spi_write()`, `spi_read()` |
| ADC | `zephyr/drivers/adc.h` | `adc_read()`, `adc_sequence_init_dt()` |
| PWM | `zephyr/drivers/pwm.h` | `pwm_set_cycles()`, `pwm_set_dt()` |
| 传感器 | `zephyr/drivers/sensor.h` | `sensor_sample_fetch()`, `sensor_channel_get()` |
| UART | `zephyr/drivers/uart.h` | `uart_poll_out()`, `uart_poll_in()`, `uart_irq_rx_enable()` |
| 线程 | `zephyr/kernel.h` | `k_thread_create()`, `K_THREAD_DEFINE()`, `k_sleep()` |
| 消息队列 | `zephyr/kernel.h` | `k_msgq_put()`, `k_msgq_get()` |
| FIFO（链表式传递） | `zephyr/kernel.h` | `k_fifo_put()`, `k_fifo_get()` |
| Work Queue | `zephyr/kernel.h` | `k_work_submit()`, `k_work_init()` |
| 定时器 | `zephyr/kernel.h` | `k_timer_start()`, `k_timer_stop()`, `k_timer_handler_set()` |
| Shell | `zephyr/shell/shell.h` | `shell_print()`, `shell_cmd_register()` |
| 日志 | `zephyr/logging/log.h` | `LOG_INF()`, `LOG_ERR()`, `LOG_DBG()`, `LOG_WRN()` |
| 设备树宏 | `zephyr/devicetree.h` | `DT_ALIAS()`, `DT_PROP()`, `DT_NODELABEL()` |
| 设备树 GPIO | `zephyr/devicetree/gpio.h` | `GPIO_DT_SPEC_GET()` — 获取 GPIO + flag 的便捷宏 |
| 文件系统 | `zephyr/fs/fs.h` | `fs_open()`, `fs_read()`, `fs_write()`, `fs_mkdir()` |

## 5.3 获取设备实例的四种方法

所有驱动 API 的第一步都是**获取设备句柄**（`const struct device *dev`），以下是四种标准写法：

```c
// 方法 1：通过 DT_ALIAS（最推荐，方便跨板移植）
// 需要在设备树中有：aliases { my-sensor = &bme280; };
const struct device *dev = DEVICE_DT_GET(DT_ALIAS(my_sensor));

// 方法 2：通过 compatible 自动匹配（只有一个设备时最简洁）
const struct device *dev = DEVICE_DT_GET_ONE(bosch_bme280);
// 注意：compatible 字符串 "bosch,bme280" → 宏中变为 bosch_bme280
// 

// 方法 3：通过 DT_NODELABEL（知道节点的 label 时）
// 设备树中有：my_uart: uart@40004400 { ... };
const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(my_uart));

// 方法 4：通过 chosen 节点（系统级资源）
const struct device *dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));

// ⚠️ 一定不要忘了检查设备是否 ready！
if (!device_is_ready(dev)) {
    printk("Device not ready!\n");
    return;
}

// ❌ 旧方法 device_get_binding("UART_1") 已弃用
// ✅ 全部改用 DEVICE_DT_GET 系列宏
```

## 5.4 查找 API 的标准化流程

```text
场景：你有一个 I2C 传感器，想读数据。

Step 1：确定子系统
  → I2C 传感器 → 可以用 I2C 驱动直接读写，也可以用 Sensor 子系统
  → 看传感器驱动的 sample 用哪个
  → 一般来说：Sensor 子系统更方便（统一 API），I2C 更底层

Step 2：找对应子系统的头文件
  ls zephyr/include/zephyr/drivers/
  → i2c.h, sensor.h, gpio.h, ...

Step 3：打开头文件，看函数声明和注释
  （Zephyr 的头文件注释质量很高，是最好的文档之一）
  grep "^__syscall\|^int\|^static" zephyr/include/zephyr/drivers/sensor.h

Step 4：找对应子系统的 sample 确认用法
  zephyr/samples/drivers/sensor/bme280/
  → 看 main.c 里的完整调用流程

Step 5：套用 sample 中的模式
  // 获取设备 → 检查 ready → 采集 → 读取
  sensor_sample_fetch(dev);
  sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &temp_val);
```

## 5.5 DT 宏的命名规律

所有设备树宏以 `DT_` 开头，理解了命名规律就能推断出宏的写法：

| 模式 | 含义 | 示例 |
|------|------|------|
| `DT_ALIAS(name)` | 别名获取节点 | `DT_ALIAS(led0)` → 节点 ID |
| `DT_CHOSEN(key)` | chosen 获取节点 | `DT_CHOSEN(zephyr_console)` |
| `DT_NODELABEL(label)` | 标签获取节点 | `DT_NODELABEL(uart0)` |
| `DT_PROP(node, prop)` | 节点属性值 | `DT_PROP(MY_NODE, reg)` |
| `DT_PROP_LEN(node, prop)` | 属性长度 | — |
| `DT_REG_ADDR(node)` | reg 的地址部分 | — |
| `DT_REG_SIZE(node)` | reg 的大小部分 | — |
| `DT_IRQN(node)` | 中断号 | — |
| `DT_INST(inst)` | 第 inst 个实例（需先定义 DT_DRV_COMPAT） | 驱动内部用 |
| `DT_INST_PROP(inst, prop)` | 实例的属性值 | 驱动内部用 |
| `DT_INST(inst, compat)` | 指定 compat 的第 inst 个实例 | 少数场景使用 |
| `DT_ENUM_IDX(node, prop)` | 枚举索引 | — |

**完整列表：** `zephyr/include/zephyr/devicetree.h`

**驱动内部的 DT_DRV_COMPAT 用法：**

```c
// 驱动源码中：
#define DT_DRV_COMPAT vendor_mydevice

// 然后就可以用 DT_INST 系列宏：
#define MY_DEVICE(inst) DEVICE_DT_INST_DEFINE(inst, ...)

// 这相当于自动为设备树中所有 compatible= vendor,mydevice 的节点
// 生成实例，从 0 开始编号
```

## 5.6 官方 Sample 的正确用法

**不要从头写 Zephyr 应用。找一个最接近的 sample 改。**

```bash
# 1. 确定子系统
ls zephyr/samples/
# basic/  drivers/  subsystems/  boards/  ...

# 2. 找最接近的
ls zephyr/samples/drivers/sensor/

# 3. 研究它的结构
tree zephyr/samples/drivers/sensor/bme280/
# ├── CMakeLists.txt
# ├── prj.conf       ← 照抄需要的 CONFIG
# ├── README.rst     ← 说明文档
# └── src
#     └── main.c     ← 复制 API 调用模式

# 4. 改成你的板子
# 改设备树 overlay、prj.conf、代码中引脚和地址
```

## 5.7 线程设计的三种常见模式

### 模式 A：单线程轮询（最简单，适合周期固定的应用）

```c
void main(void)
{
    const struct device *sensor = DEVICE_DT_GET_ONE(bosch_bme280);
    const struct device *led = DEVICE_DT_GET(DT_ALIAS(led0));

    if (!device_is_ready(sensor) || !device_is_ready(led)) {
        return;
    }

    while (1) {
        /* 采集 */
        sensor_sample_fetch(sensor);
        /* 处理并控制 */
        sensor_channel_get(sensor, SENSOR_CHAN_AMBIENT_TEMP, &val);
        gpio_pin_set(led, pin, val.float_val > 30.0);
        /* 等待 */
        k_sleep(K_MSEC(1000));
    }
}
```

### 模式 B：生产者-消费者（双线程，适合需要缓冲解耦的场景）

```c
/* 定义消息队列：存 10 个 float */
K_MSGQ_DEFINE(sensor_msgq, sizeof(float), 10, 4);

/* 生产者线程：采集 */
void sensor_thread(void *, void *, void *)
{
    const struct device *sensor = DEVICE_DT_GET_ONE(bosch_bme280);
    struct sensor_value temp;
    while (1) {
        sensor_sample_fetch(sensor);
        sensor_channel_get(sensor, SENSOR_CHAN_AMBIENT_TEMP, &temp);
        k_msgq_put(&sensor_msgq, &temp.val1, K_FOREVER);
        k_sleep(K_MSEC(100));
    }
}

/* 消费者线程：根据数据控制执行器 */
void control_thread(void *, void *, void *)
{
    float temp;
    while (1) {
        k_msgq_get(&sensor_msgq, &temp, K_FOREVER);
        gpio_pin_set(led_dev, pin, temp > 30.0f);
    }
}

K_THREAD_DEFINE(sensor_tid, 1024, sensor_thread, NULL, NULL, NULL, 5, 0, 0);
K_THREAD_DEFINE(control_tid, 1024, control_thread, NULL, NULL, NULL, 5, 0, 0);

void main(void) { /* 初始化即可，线程会自动运行 */ }
```

### 模式 C：中断 + Work Queue（适合响应外部事件）

```c
static struct gpio_callback button_cb_data;
static struct k_work button_work;

/* Work handler — 在系统 work queue 中执行 */
static void button_work_handler(struct k_work *work)
{
    /* 这里可以做更多工作，不会阻塞中断 */
    gpio_pin_toggle(led_dev, led_pin);
    printk("Button pressed!\n");
}

/* 中断回调 — 只做最少的事 */
void button_pressed(const struct device *dev,
                    struct gpio_callback *cb, uint32_t pins)
{
    k_work_submit(&button_work);  /* 把实际工作交给 work queue */
}

void main(void)
{
    /* 初始化 work */
    k_work_init(&button_work, button_work_handler);

    /* 配置 GPIO 中断 */
    gpio_pin_configure(button_dev, button_pin, GPIO_INPUT);
    gpio_pin_interrupt_configure(button_dev, button_pin,
                                 GPIO_INT_EDGE_TO_ACTIVE);
    gpio_init_callback(&button_cb_data, button_pressed, BIT(button_pin));
    gpio_add_callback(button_dev, &button_cb_data);

    while (1) { k_sleep(K_FOREVER); }
}
```

---

# 第六章：实战案例——从零做一个外设驱动

用完整案例串联前五章讲的流程：**写 binding → 设备树 → prj.conf → 应用代码 → 调试验证。**

## 6.1 场景设定

- **板子：** NUCLEO-F411RE（STM32F411）
- **外设：** I2C 温湿度传感器 AHT20（假设 Zephyr 没有现成驱动）
- **需求：** 每秒读一次温湿度，通过串口打印

## 6.2 完整开发流程

### Step 1：确定芯片和 compatible

```bash
# 先查 Zephyr 有没有现成驱动
find zephyr/dts/bindings -name "*.yaml" | xargs grep -l "aht\|aosong"
# 没有 → 我们要自己写

# 查 datasheet 确定：
# 厂商 = aosong
# 芯片 = aht20
# I2C 地址 = 0x38
# → compatible = "aosong,aht20"
```

### Step 2：写 Binding YAML

```yaml
# 文件：my_app/dts/bindings/aosong,aht20.yaml
# 放在应用目录下，Zephyr 会自动搜索

compatible: "aosong,aht20"

include: [i2c-device.yaml]

description: |
  Aosong AHT20 temperature and humidity sensor.

properties:
  alert-gpios:
    type: phandle-array
    required: false
    description: |
      Optional alert GPIO pin.
```

### Step 3：写设备树 Overlay

```devicetree
// my_app/boards/nucleo_f411re.overlay

&i2c1 {
    status = "okay";
    pinctrl-0 = <&i2c1_default>;
    pinctrl-names = "default";

    aht20@38 {
        compatible = "aosong,aht20";
        reg = <0x38>;
        label = "AHT20";
    };
};
```

**验证 overlay 是否生效：**

```bash
west build -b nucleo_f411re . -t devicetree
# ↓ 查看合并后的 dts，搜索 "aht20"
grep -A5 "aht20" build/zephyr/zephyr.dts
```

### Step 4：编辑 prj.conf

```
# my_app/prj.conf

# I2C 驱动
CONFIG_I2C=y
CONFIG_I2C_STM32=y

# 串口控制台
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y
CONFIG_PRINTK=y

# 日志
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3

# Shell（可选，调试用）
CONFIG_SHELL=y
```

### Step 5：写 C 代码

```c
// my_app/src/main.c

#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/i2c.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(aht20_demo, LOG_LEVEL_INF);

#define AHT20_I2C_ADDR      0x38
#define AHT20_INIT_CMD      0xBE
#define AHT20_TRIGGER_CMD   0xAC
#define AHT20_SOFTRESET_CMD 0xBA

static int aht20_init(const struct device *i2c_dev)
{
    uint8_t reset = AHT20_SOFTRESET_CMD;
    int ret;

    ret = i2c_write(i2c_dev, &reset, 1, AHT20_I2C_ADDR);
    if (ret) { return ret; }
    k_sleep(K_MSEC(20));

    uint8_t init[] = {AHT20_INIT_CMD, 0x08, 0x00};
    ret = i2c_write(i2c_dev, init, 3, AHT20_I2C_ADDR);
    if (ret) { return ret; }
    k_sleep(K_MSEC(10));

    LOG_INF("AHT20 initialized");
    return 0;
}

static int aht20_read(const struct device *i2c_dev,
                      float *temp, float *hum)
{
    uint8_t trigger[] = {AHT20_TRIGGER_CMD, 0x33, 0x00};
    uint8_t raw[7];
    int ret;

    ret = i2c_write(i2c_dev, trigger, 3, AHT20_I2C_ADDR);
    if (ret) { return ret; }
    k_sleep(K_MSEC(100));

    ret = i2c_read(i2c_dev, raw, 6, AHT20_I2C_ADDR);
    if (ret) { return ret; }

    /* 解析温湿度（20 位数据） */
    uint32_t hum_raw = ((uint32_t)raw[1] << 12) |
                       ((uint32_t)raw[2] << 4) |
                       ((uint32_t)raw[3] >> 4);
    uint32_t temp_raw = ((uint32_t)raw[3] & 0x0F) << 16 |
                        (uint32_t)raw[4] << 8 |
                        (uint32_t)raw[5];

    *hum = (float)hum_raw * 100.0f / 1048576.0f;
    *temp = (float)temp_raw * 200.0f / 1048576.0f - 50.0f;

    return 0;
}

void main(void)
{
    const struct device *i2c_dev = DEVICE_DT_GET(DT_NODELABEL(i2c1));
    float temperature, humidity;

    if (!device_is_ready(i2c_dev)) {
        LOG_ERR("I2C device not ready");
        return;
    }

    if (aht20_init(i2c_dev) != 0) {
        LOG_ERR("AHT20 init failed");
        return;
    }

    while (1) {
        if (aht20_read(i2c_dev, &temperature, &humidity) == 0) {
            printk("Temp: %.1f C, Hum: %.1f %%\n",
                   temperature, humidity);
        }
        k_sleep(K_SECONDS(1));
    }
}
```

### Step 6：CMakeLists.txt

```cmake
# my_app/CMakeLists.txt
cmake_minimum_required(VERSION 3.20.0)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_app)

FILE(GLOB sources src/*.c)
target_sources(app PRIVATE ${sources})
```

### Step 7：编译和调试

```bash
# 编译
cd my_app
west build -b nucleo_f411re .

# 检查合并设备树
grep -A5 "aht" build/zephyr/zephyr.dts

# 检查 config
grep "I2C" build/zephyr/.config

# 检查生成的 DT 宏
grep -i "aht" build/zephyr/include/generated/devicetree_generated.h

# 烧录
west flash

# 查看串口输出
west build -t monitor
# 或
minicom -D /dev/ttyACM0 -b 115200
```

---

# 附录：文件索引与速查

## A. 关键路径速查

| 你要什么 | 去哪儿找 |
|---------|---------|
| 设备树绑定 YAML | `zephyr/dts/bindings/<类别>/*.yaml` |
| SoC 级设备树 | `zephyr/dts/<架构>/<厂商>/` |
| 板级设备树 | `zephyr/boards/<厂商>/<板名>/*.dts` |
| 板级 defconfig | `zephyr/boards/<厂商>/<板名>/*_defconfig` |
| 驱动源码 | `zephyr/drivers/<类别>/` |
| 头文件（API） | `zephyr/include/zephyr/` |
| 设备树 API 宏全集 | `zephyr/include/zephyr/devicetree.h` |
| GPIO 标志位宏 | `zephyr/include/zephyr/dt-bindings/gpio/gpio.h` |
| Pinctrl 宏（芯片系列） | `zephyr/include/zephyr/dt-bindings/pinctrl/<芯片>.h` |
| Kconfig 源码定义 | `zephyr/**/Kconfig*`（grep 大法） |
| 示例代码 | `zephyr/samples/` |
| 官方文档（RST） | `zephyr/doc/` |
| 最终硬件描述 | `build/zephyr/zephyr.dts` |
| 最终配置 | `build/zephyr/.config` |
| 生成 DT 宏 | `build/zephyr/include/generated/devicetree_generated.h` |

## B. 常用调试命令速查

```bash
# === 调试设备树 ===
west build -t devicetree          # 验证设备树正确性
cat build/zephyr/zephyr.dts       # 查看合并结果
grep "gpio0" build/zephyr/zephyr.dts  # 搜索特定节点

# === 调试 Kconfig ===
west build -t menuconfig          # 交互式配置查看
cat build/zephyr/.config          # 最终配置
grep "CONFIG_I2C" build/zephyr/.config

# === 调试构建 ===
west build -t pristine            # 清空并重新构建
west build -v                     # verbose 输出
west build -p auto                # 增量构建（自动检测变化）

# === 查看运行时 ===
west build -t monitor             # 串口监控
west flash                        # 烧录（自动识别调试器）

# === STM32 特有 ===
west flash --runner stlink        # 强制使用 ST-Link
west flash --runner openocd       # 使用 OpenOCD
STM32_Programmer_CLI -c port=SWD -w build/zephyr/zephyr.bin 0x08000000  # 备选烧录
# 查看 STM32 外设地址映射
cat build/zephyr/zephyr.dts | grep "reg = <" | grep "\#include>" 

# === 查看生成的宏 ===
cat build/zephyr/include/generated/devicetree_generated.h | head -50
```

## C. 常见问题排查清单

```
问题：设备没有工作
  ├─ 设备树是否 enable？
  │   grep "status" build/zephyr/zephyr.dts | grep "mydevice"
  ├─ 驱动是否编译进去了？
  │   grep "CONFIG_xxx" build/zephyr/.config
  │   （注意看 =y 还是 is not set）
  ├─ 代码里有没有检查 device_is_ready()？
  ├─ 引脚配置是否正确？用 GPIO 的话确认 pinctrl
  └─ 🎯 STM32 特有：检查 RCC 时钟配置？
      grep "status" build/zephyr/zephyr.dts | grep "clk_\|pll"
      时钟源没配好，外设不会工作

问题：找不到某个 CONFIG 选项
  ├─ menuconfig 搜索（先 west build -t menuconfig，再按 /）
  ├─ grep -r "config XXX" zephyr/ --include="Kconfig*"
  └─ 注意：搜索 "config XXX" 不是 "CONFIG_XXX"

问题：设备树属性不生效
  ├─ cat build/zephyr/zephyr.dts 查看合并结果
  ├─ 确认 overlay 被正确加载了
  └─ 确认 compatible 字符串完全匹配

问题：API 不知道用哪个
  ├─ 找对应子系统的头文件（注释是最好的文档）
  ├─ 找对应子系统的 sample
  └─ 看 zephyr/doc/ 中的 RST 文档

问题：编译报错 "undefined DT_xxx"
  ├─ 确认设备树中有对应的节点
  ├─ 确认节点 status = "okay"
  └─ 确认宏名拼写正确（注意 - 变 _）
```

## D. 推荐学习路线

```
第 1 周：跑通官方 sample
  ├─ 读懂 blinky：设备树(GPIO LED) → prj.conf → main.c
  ├─ 跑通 hello_world → 确认串口输出
  └─ 跑通 button → 理解 GPIO 输入和中断

第 2 周：改自己板子的设备树
  ├─ 添加新的 GPIO LED（改 overlay）
  ├─ 添加 I2C 传感器（用现有 binding）
  └─ 用 menuconfig 看配置变化

第 3 周：写多线程应用
  ├─ 用线程 + 消息队列解耦采集和控制
  ├─ 用 work queue 处理中断事件
  └─ 用 Shell 子系统加调试命令

第 4 周：写自己的驱动
  ├─ 写 binding yaml
  ├─ 参考现有驱动代码
  └─ 调试设备树 + 驱动配合
```

## E. 速记口诀

```
设备树管硬件，Kconfig管功能，C代码管逻辑。
属性名找binding，参数名搜Kconfig，API名去sample。
dts/chosen/alias → 设备获取三板斧。
编译先用west build -t devicetree → 有问题早暴露。
```

---

> **最后记住：** Zephyr 的代码和文档本身是最好的学习资料。官方 sample 解决了 80% 的"第一步怎么做"的问题。剩下的 20% 是在具体报错时逐步调试查资料解决的。别怕读源码——源码才是最终真理。🦞
