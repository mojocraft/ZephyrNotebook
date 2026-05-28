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
- [第六章：自定义电路板——Zephyr 如何支持一块新板子？](#第六章自定义电路板zephyr-如何支持一块新板子)
- [第七章：实战案例——从零做一个外设驱动](#第七章实战案例从零做一个外设驱动)
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

**如何找到 GPIO 控制器名（以 STM32 为例）：**

```bash
grep -r "gpio@" dts/arm/st/f4/stm32f405.dtsi
# 输出类似：
# gpio_a: gpio@40020000 { ... };
# gpio_b: gpio@40020400 { ... };
```

> 🎯 **换成你的芯片怎么办？** 把架构和厂商路径换成你的：
> ```bash
> # 确定你的芯片架构
> ls dts/<架构>/<厂商>/
> # 在对应的 SoC dtsi 里搜 gpio@
> grep "gpio@" dts/<架构>/<厂商>/<你的芯片>.dtsi
> ```

### Pinctrl 节点——不同芯片写法不同

Pinctrl（引脚复用）的定义方式**因芯片厂商而异**，不是统一的写法。

**STM32 风格：** 使用 `psels` 属性 + `STM32_PSEL_xxx` 宏
```devicetree
&pinctrl {
    uart0_default: uart0_default {
        group1 {
            psels = <STM32_PSEL_UART_TX(PB6)>;
        };
    };
};
```

**NXP i.MX RT 风格：** 使用 `group` + 引脚名直接指定
```devicetree
&iomuxc {
    pinctrl_uart1: uart1grp {
        fsl,pins = <
            MXRT_PAD_GPIO_AD_B0_12__UART1_TXD 0x1e0
            MXRT_PAD_GPIO_AD_B0_13__UART1_RXD 0x1e0
        >;
    };
};
```

**Nordic nRF 风格：** 直接在设备树外设节点中配引脚
```devicetree
&uart0 {
    tx-pin = <6>;   rx-pin = <8>;
    rts-pin = <5>;  cts-pin = <7>;
};
```

> 💡 **怎么查你的芯片用哪种写法？**
> ```bash
> # 找一个你芯片的官方板子，看它的 .dts 里 pinctrl 怎么写
> cat boards/<厂商>/<板名>/*.dts | grep -A5 "pinctrl"
> # 或者搜芯片 pinctrl 头文件
> ls include/zephyr/dt-bindings/pinctrl/
> ```

Pinctrl 宏定义位置：
```
zephyr/include/zephyr/dt-bindings/pinctrl/<芯片系列>.h
```
例如 STM32 系列是 `stm32f4-pinctrl.h`，NXP 是 `imxrt-pinctrl.h`。

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

### 时钟配置——不同芯片家族的做法

> 💡 **为什么上一版文档特意说 "不在 Kconfig 里"？**
> 因为很多芯片恰恰**是**在 Kconfig 里配时钟的。如果不说明，读者可能会用 STM32 的经验套到所有芯片上，然后发现找不到选项。

Zephyr 没有统一规定时钟放哪——每个 SoC 系列的维护者自己决定。

**目前主要有两种路线：**

| 配置方式 | 代表芯片 | 怎么看当前值 |
|---------|---------|------------|
| **设备树配时钟** | STM32、Microchip SAM | `cat build/zephyr/zephyr.dts` 搜 clk_/pll |
| **Kconfig 配时钟** | NXP i.MX RT / Kinetis、Nordic nRF | `cat build/zephyr/.config` 搜 CLOCK |
| 混合模式 | 部分 SoC 两者都有 | 两种都查一下 |

---

#### ① 设备树路线：STM32

STM32 的时钟节点全部定义在 SoC 级 .dtsi 中，板级 .dts 或 overlay 中修改：

```devicetree
&clk_hse {
    hse-clk = <8000000>;
    hse-ready-hsi = <1>;
    status = "okay";
};
&pll {
    pll-mul = <168>;
    pll-div = <4>;
    pll-p = <2>;
    pll-q = <7>;
    status = "okay";
};
&rcc {
    clocks = <&pll>;
    clock-frequency = <DT_HZ_168000000>;
};
```

**查看你的 STM32 芯片有哪些时钟节点：**
```bash
grep "clk_" zephyr/dts/arm/st/f4/stm32f405.dtsi | head -20
# → clk_hse, clk_lse, pll 等
```

---

#### ② Kconfig 路线：NXP i.MX RT

NXP 的时钟配置在 Kconfig 里，不在设备树中：

```bash
# 时钟源、PLL 分频等都走 Kconfig
grep -r "CLOCK" zephyr/soc/arm/nxp_imx/rt/ --include="Kconfig*"
# → CLOCK_ACMP, CLOCK_ARM 等选项
```

#### ③ Kconfig 路线：Nordic nRF

```bash
# 低频时钟源选择在 Kconfig
CONFIG_CLOCK_CONTROL_NRF_K32SRC_XTAL=y   # 用外部晶振
CONFIG_CLOCK_CONTROL_NRF_K32SRC_RC=y     # 用内部 RC
```

---

**快速判断法：你的芯片走哪条路？**

```bash
# 方法 1：在设备树里搜时钟相关节点
grep -r "clk_\|clock.*=" zephyr/dts/<架构>/<厂商>/ --include="*.dtsi" | head -10

# 方法 2：在 Kconfig 里搜 CLOCK 选项
grep -r "config CLOCK" zephyr/soc/<架构>/<厂商>/ --include="Kconfig*" | head -10

# 方法 3：看官方板子——dts 和 defconfig 都查
head -30 boards/<厂商>/<板名>/*_defconfig | grep -i clock
head -50 boards/<厂商>/<板名>/*.dts | grep -i clk
```

> 💡 **记忆法：** 看到 `&clk_hse { ... };` 是设备树路线，看到 `CONFIG_CLOCK_xxx=y` 是 Kconfig 路线。

### Flash 分区（MCUboot 或 OTA 时需要）

```devicetree
/ {
    chosen {
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
            reg = <0x00000000 0x00010000>;
        };
        slot0_partition: partition@10000 {
            label = "image-0";
            reg = <0x00010000 0x00060000>;
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
  My custom device.

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
| `i2c-device.yaml` | `reg`（I2C 地址）| I2C 外设 |
| `spi-device.yaml` | `reg`（CS 号）| SPI 外设 |
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

# 方法 3：查看最终合并的设备树
cat build/zephyr/zephyr.dts | grep -A 10 "mydevice"
```

---

# 第四章：prj.conf——参数名从哪来？

## 4.1 Kconfig 参数定义的核心逻辑

每个 `CONFIG_XXX` 都在 Zephyr 源码树**某个地方被定义了**。定义它的位置决定了它的用途。

**Kconfig 文件分布（按查找优先级排序）：**

```
zephyr/Kconfig                       → 全局、内核基础
zephyr/drivers/<类别>/Kconfig        → 驱动使能
zephyr/subsys/<子系统>/Kconfig       → 子系统（Shell、Log…）
zephyr/modules/<模块>/Kconfig        → 外部模块
boards/<厂商>/<板名>/Kconfig.defconfig → 板级默认值
arch/<架构>/Kconfig                  → 架构相关
```

## 4.2 五种查找方法（逐个掌握）

### 方法 1：menuconfig（首选）

```bash
west build -t menuconfig   # 终端界面
# 按 / 键搜索 → 按 ? 看帮助 → 按 1/2/3 跳转
```

### 方法 2：grep 搜索

```bash
# 注意：搜的是 "config XXX" 不是 "CONFIG_XXX"
grep -r "config I2C_STM32" zephyr/ --include="Kconfig*"
```

### 方法 3：看硬件板子的 defconfig

```bash
cat zephyr/boards/arm/nucleo_f401re/nucleo_f401re_defconfig
```

### 方法 4：看官方 sample 的 prj.conf

```bash
cat zephyr/samples/drivers/gpio/prj.conf
cat zephyr/samples/sensor/bme280/prj.conf
```

### 方法 5：查看编译结果

```bash
cat build/zephyr/.config | grep -E "(I2C|UART|SHELL)" | head -20
```

## 4.3 完整追踪示例：如何追踪任意芯片的驱动名

方法是一样的，方法 2 的 grep 换成你的芯片名搜就行：

```bash
# 以 STM32 的 I2C 驱动为例：
# 想知道 CONFIG_I2C_STM32 从哪来？

# Step 1：搜定义
grep -r "config I2C_STM32" zephyr/ --include="Kconfig*"
# 结果在：zephyr/drivers/i2c/Kconfig.stm32

# Step 2：打开看
cat zephyr/drivers/i2c/Kconfig.stm32 | head -30
# config I2C_STM32
#     bool "STM32 I2C driver"
#     depends on I2C && $(dt_compat_enabled,...)

# Step 3：理解依赖
# "depends on I2C" → 必须先有 CONFIG_I2C=y
# 设备树检查 → 设备树中有 compatible = "st,stm32-i2c" 的节点

# Step 4：你在 prj.conf 里只需要写
CONFIG_I2C=y
CONFIG_I2C_STM32=y
```

> 💡 **换一个芯片也是一样的流程：**
> ```bash
> # 比如你是 NXP i.MX RT，搜 I2C 对应的 Kconfig
> grep -r "config I2C" zephyr/drivers/i2c/ --include="Kconfig*" | grep -i "imx\\|nxp"
> # → CONFIG_I2C_MCUX=y（MCUXpresso SDK 的 I2C 驱动）
> # → prj.conf 里写 CONFIG_I2C=y CONFIG_I2C_MCUX=y
> ```

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

### 具体驱动——不同芯片的驱动名不同

**不要照抄 STM32 的名字。** 用同样的方法找你芯片的选项：

| 你的芯片 | GPIO 驱动名 | I2C 驱动名 |
|---------|------------|-----------|
| STM32 | `CONFIG_GPIO_STM32=y` | `CONFIG_I2C_STM32=y` |
| NXP i.MX RT | `CONFIG_GPIO_MCUX_IGPIO=y` | `CONFIG_I2C_MCUX=y` |
| Nordic nRF | `CONFIG_GPIO_NRF=y` | `CONFIG_I2C_NRF_TWIM=y` |
| ESP32 | `CONFIG_GPIO_ESP32=y` | `CONFIG_I2C_ESP32=y` |

> 🎯 **不确定你的芯片驱动叫什么？**
> ```bash
> # 找 GPIO 驱动：grep -r "config GPIO" zephyr/drivers/gpio/ --include="Kconfig*" | grep -i "你的芯片"
> # 找 I2C 驱动：grep -r "config I2C" zephyr/drivers/i2c/ --include="Kconfig*" | grep -i "你的芯片"
> # 或者看官方板子的 defconfig
> cat boards/<厂商>/<板名>/*_defconfig | grep -E "(GPIO|I2C|SPI)"
> ```

传感器驱动是跨芯片的（走总线），各芯片通用：
```
CONFIG_SENSOR_BME280=y          # BME280 传感器驱动
```

### 系统功能

```
CONFIG_SHELL=y                  # Shell 命令行
CONFIG_LOG=y                    # 日志系统
CONFIG_ASSERT=y                 # 断言（开发时开）
CONFIG_PRINTK=y                 # printk 输出
CONFIG_UART_CONSOLE=y           # UART 控制台
```

### 线程/内存

```
CONFIG_MAIN_STACK_SIZE=4096     # main 线程栈大小
CONFIG_HEAP_MEM_POOL_SIZE=4096  # 堆大小
```

### 调试

```
CONFIG_DEBUG=y                  # 调试模式
CONFIG_LOG_DEFAULT_LEVEL=4      # 日志级别
CONFIG_SHELL_BACKEND_SERIAL=y   # Shell 串口后端
```

---

# 第五章：应用开发——我怎么知道该调哪个 API？

## 5.1 核心思维：先选子系统，再找 API

```
我要控制 LED
   └─ GPIO → #include <zephyr/drivers/gpio.h>
       └─ gpio_pin_configure(), gpio_pin_set(), gpio_pin_toggle()

我要读温度传感器
   └─ Sensor → #include <zephyr/drivers/sensor.h>
       └─ sensor_sample_fetch(), sensor_channel_get()

我要串口交互
   ├─ 简单打印 → printk()
   └─ 交互式命令 → Shell 子系统 → CONFIG_SHELL=y
```

## 5.2 子系统 × 头文件 × API 查询表

| 你要做什么 | 头文件 | 常用 API |
|-----------|--------|---------|
| GPIO | `drivers/gpio.h` | `gpio_pin_configure/set/get/interrupt_configure` |
| I2C | `drivers/i2c.h` | `i2c_write_read/burst_read/transfer` |
| SPI | `drivers/spi.h` | `spi_transceive/write/read` |
| ADC | `drivers/adc.h` | `adc_read/sequence_init_dt` |
| PWM | `drivers/pwm.h` | `pwm_set_cycles/set_dt` |
| 传感器 | `drivers/sensor.h` | `sensor_sample_fetch/channel_get` |
| UART | `drivers/uart.h` | `uart_poll_out/poll_in/irq_rx_enable` |
| 线程 | `kernel.h` | `k_thread_create, K_THREAD_DEFINE, k_sleep` |
| 消息队列 | `kernel.h` | `k_msgq_put/get` |
| Work Queue | `kernel.h` | `k_work_submit/init` |
| 定时器 | `kernel.h` | `k_timer_start/stop/handler_set` |
| Shell | `shell/shell.h` | `shell_print, shell_cmd_register` |
| 日志 | `logging/log.h` | `LOG_INF/ERR/DBG/WRN` |
| 文件系统 | `fs/fs.h` | `fs_open/read/write/mkdir` |

## 5.3 获取设备实例的四种方法

```c
// 方法 1：通过 DT_ALIAS（最推荐，方便跨板移植）
const struct device *dev = DEVICE_DT_GET(DT_ALIAS(my_sensor));

// 方法 2：通过 compatible 自动匹配
const struct device *dev = DEVICE_DT_GET_ONE(bosch_bme280);

// 方法 3：通过 DT_NODELABEL
const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(i2c1));

// 方法 4：通过 chosen 节点
const struct device *dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));

// ⚠️ 一定不要忘了检查设备是否 ready！
if (!device_is_ready(dev)) {
    printk("Device not ready!\n");
    return;
}
```

## 5.4 查找 API 的标准化流程

```
场景：你有一个 I2C 传感器，想读数据。

Step 1：确定子系统 → I2C 或 Sensor
Step 2：找头文件 → ls zephyr/include/zephyr/drivers/
Step 3：打开头文件看注释（Zephyr 头文件注释质量很高）
Step 4：找对应子系统的 sample
  → zephyr/samples/drivers/sensor/bme280/main.c
Step 5：套用模式
  sensor_sample_fetch(dev);
  sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &temp_val);
```

## 5.5 DT 宏的命名规律

| 模式 | 含义 |
|------|------|
| `DT_ALIAS(name)` | 别名获取节点 |
| `DT_CHOSEN(key)` | chosen 获取节点 |
| `DT_NODELABEL(label)` | 标签获取节点 |
| `DT_PROP(node, prop)` | 节点属性值 |
| `DT_REG_ADDR/SIZE(node)` | reg 地址/大小 |
| `DT_IRQN(node)` | 中断号 |
| `DT_INST_PROP(inst, prop)` | 驱动内部实例属性 |

驱动内部用法：

```c
#define DT_DRV_COMPAT vendor_mydevice
// 自动遍历所有 compatible= vendor,mydevice 的节点
```

## 5.6 官方 Sample 的正确用法

```bash
# 确定子系统 → 找最接近的 sample → 研究结构 → 改成你的板子
tree zephyr/samples/drivers/sensor/bme280/
# ├── CMakeLists.txt
# ├── prj.conf       ← 照抄需要的 CONFIG
# ├── README.rst     ← 说明文档
# └── src/main.c     ← 复制 API 调用模式
```

## 5.7 线程设计的三种常见模式

### 模式 A：单线程轮询（最简单）

```c
void main(void)
{
    while (1) {
        sensor_sample_fetch(sensor);
        sensor_channel_get(sensor, SENSOR_CHAN_AMBIENT_TEMP, &val);
        gpio_pin_set(led, pin, val.float_val > 30.0);
        k_sleep(K_MSEC(1000));
    }
}
```

### 模式 B：生产者-消费者（双线程）

```c
K_MSGQ_DEFINE(sensor_msgq, sizeof(float), 10, 4);

void sensor_thread(void *, void *, void *)
{
    /* 采集 → k_msgq_put */
}

void control_thread(void *, void *, void *)
{
    /* k_msgq_get → 控制 */
}

K_THREAD_DEFINE(sensor_tid, 1024, sensor_thread, NULL, NULL, NULL, 5, 0, 0);
K_THREAD_DEFINE(control_tid, 1024, control_thread, NULL, NULL, NULL, 5, 0, 0);
```

### 模式 C：中断 + Work Queue

```c
void button_pressed(const struct device *dev,
                    struct gpio_callback *cb, uint32_t pins)
{
    k_work_submit(&button_work);  /* 只提交，不做实际工作 */
}

void main(void)
{
    k_work_init(&button_work, button_work_handler);
    gpio_pin_interrupt_configure(button_dev, button_pin,
                                 GPIO_INT_EDGE_TO_ACTIVE);
    while (1) { k_sleep(K_FOREVER); }
}
```

---

# 第六章：自定义电路板——Zephyr 如何支持一块新板子？

做完外设驱动后，下一步就是你的**自定义硬件**了。Zephyr 板级支持对新手来说最困惑的地方是：文件多、分布广、不知道去哪找参考。

## 6.1 什么时候需要自定义板子？

| 场景 | 怎么做 | 难度 |
|------|--------|------|
| 官方开发板 + 简单外设 | 只用 overlay，不需要自定义板子 | ⭐ |
| 自己画的 PCB，MCU 型号 Zephyr 支持 | 基于相近官方板子修改 | ⭐⭐ |
| 自己画的 PCB，有 SoC 支持但没板级文件 | 从零写 board 支持 | ⭐⭐⭐ |
| 全新的 MCU 系列 | 写 SoC 支持 + board 支持 | ⭐⭐⭐⭐⭐ |

**本文聚焦：** MCU 型号 Zephyr 已经有 SoC 级支持（dtsi + 驱动），但没你的板子。

> 💡 **先问自己：Zephyr 支持我的 MCU 吗？**
> ```bash
> find zephyr/dts -name "*.dtsi" | xargs grep -l "stm32f411"
> # 有 → SoC 支持存在，只需要写板级文件
> # 没有 → 需要从 SoC 层面开始（本文不展开）
> ```

## 6.2 板级目录结构——一个板子需要哪些文件？

以官方板子 `nucleo_f411re` 为例：

```
boards/arm/nucleo_f411re/
├── nucleo_f411re.dts          ← 板级设备树（核心！）
├── nucleo_f411re.yaml         ← 板子元数据
├── nucleo_f411re_defconfig    ← 默认 Kconfig 配置
├── Kconfig.defconfig          ← 板级 Kconfig 默认值
├── board.cmake                ← 烧录/调试器配置
├── CMakeLists.txt             ← 构建定义
├── doc/img/                   ← 板子图片
├── doc/index.rst              ← 板子文档
└── pre_dt_board.cmake         ← 构建设置（很少用）
```

**最少需要哪些文件才能编译通过？**

```
myboard/
├── myboard.dts              # ★ 必须：板级设备树
├── myboard.yaml             # ★ 必须：板子描述
├── myboard_defconfig        # ★ 必须：默认配置
├── Kconfig.defconfig        # ☆ 推荐
├── board.cmake              # ☆ 推荐
└── CMakeLists.txt           # ☆ 推荐
```

## 6.3 Board DTS——板级设备树怎么写？

> ⚠️ **以下示例基于 STM32。** 不同芯片的 `#include` 路径、外设节点名、pinctrl 写法都不同，不要照抄。关键是理解**结构**，具体名称以你芯片的 `.dtsi` 为准。

### Board DTS 的标准结构（以 STM32 为例）

```devicetree
/* boards/<arch>/<vendor>/<board>.dts */

/* 1. 包含 SoC 级设备树（含有所有片上外设定义）*/
/* ★ 换成你的芯片路径：看同系列官方板子的 #include */
#include <st/f4/stm32f411Xe.dtsi>
/* 2. 包含芯片系列共有的 pinctrl 定义 */
#include <st/f4/stm32f4-pinctrl.dtsi>

/ {
    model = "My Custom Board";
    compatible = "myvendor,myboard", "st,stm32f411";

    /* 3. chosen 节点：指定系统级资源 */
    chosen {
        zephyr,console = &usart2;
        zephyr,shell-uart = &usart2;
        zephyr,sram = &sram0;
        zephyr,flash = &flash0;
    };

    /* 4. 板级外设（如 LED、按键）*/
    leds {
        compatible = "gpio-leds";
        green_led: led_0 {
            gpios = <&gpioa 5 GPIO_ACTIVE_HIGH>;
            label = "User LD2";
        };
    };

    gpio_keys {
        compatible = "gpio-keys";
        user_button: button {
            gpios = <&gpioc 13 GPIO_ACTIVE_LOW>;
        };
    };

    /* 5. 别名（方便应用移植）*/
    aliases {
        led0 = &green_led;
        sw0 = &user_button;
    };
};

/* 6. 使能片上外设并配置引脚 */
&usart2 {
    pinctrl-0 = <&usart2_tx_pa2 &usart2_rx_pa3>;
    pinctrl-names = "default";
    current-speed = <115200>;
    status = "okay";
};

&i2c1 {
    pinctrl-0 = <&i2c1_scl_pb8 &i2c1_sda_pb9>;
    pinctrl-names = "default";
    status = "okay";
};
```

> 🎯 **换成你的芯片，怎么知道 #include 什么？**
> ```bash
> # 先确定你的芯片在 zephyr/dts 下的路径
> find zephyr/dts -name "*你的芯片*.dtsi"
> # 例如：zephyr/dts/arm/nxp/nxp_rt1050.dtsi
> # 然后看同系列官方板子的 .dts 文件开头
> head -10 boards/<厂商>/<板名>/*.dts
> ```

### Board DTS 核心要素

| 组成部分 | 作用 | 必须？ | 参考来源 |
|---------|------|--------|---------|
| `#include` SoC dtsi | 引入片上外设定义 | ✅ | 同系列 SoC 的 .dtsi |
| `model` / `compatible` | 板子标识符 | ✅ | 命名规范见下文 |
| `chosen` | 指定 console/shell/flash | ✅ | 官方板子抄 |
| `aliases` | 设备别名 | ✅ | 便于应用移植 |
| `leds` / `gpio_keys` | 板级外设节点 | 推荐 | 参考同类板子 |
| `&uart1 { ... }` | 使能外设、配引脚 | ✅ | 看 datasheet 引脚表 |

### compatible 命名规范

```
compatible = "st,stm32f411re-nucleo", "st,stm32f411";
# 格式：<厂商>,<板名>, <厂商>,<SoC型号>
# 顺序：从具体到通用
```

> 📚 **参考来源：** 找跟你 MCU 最像的官方板子直接复制改
> ```bash
> find boards/ -name "*.dts" | xargs grep -l "你的芯片型号"
> ```

## 6.4 Board YAML——板子元数据

```yaml
# boards/<arch>/<vendor>/<board>.yaml
description: My Custom Board
identifier: myboard             # 与目录/文件名一致
name: MyCustomBoard
type: mcu
arch: arm
rom_size: 524288               # Flash 大小（bytes）
ram_size: 131072               # SRAM 大小（bytes）
supported:
  - stlink
  - jlink
  - pyocd
toolchain:
  - zephyr
  - gnuarmemb
features:
  - gpio
  - i2c
  - spi
  - usart
```

> 💡 **不确定参数值？**
> - `arch` → 看你 MCU 的内核架构（Cortex-M → arm，RISC-V → riscv）
> - `rom_size` / `ram_size` → datasheet 里 Flash/SRAM 大小
> - 不知道就找一个同系列官方板子的 yaml 直接抄

## 6.5 defconfig 和 Kconfig.defconfig

### defconfig（必须）

```bash
# boards/<arch>/<vendor>/<board>_defconfig
# ★ 以下为 STM32 示例，换成你的芯片对应的 CONFIG_SOC_xxx
CONFIG_SOC_STM32F411XE=y         # SoC 型号
CONFIG_SOC_SERIES_STM32F4X=y    # SoC 系列
CONFIG_ARM=y                     # 架构
CONFIG_CORTEX_M_SYSTICK=y       # SysTick
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y
CONFIG_PRINTK=y
```

> 🎯 **如何找到正确的 CONFIG_SOC_xxx？**
> ```bash
> grep -r "SOC_你的芯片型号" zephyr/ --include="Kconfig*"
> # 或者看同系列官方板子的 defconfig
> ```

### Kconfig.defconfig（推荐）

```python
# boards/<arch>/<vendor>/Kconfig.defconfig
if SOC_STM32F411XE

config SOC
    default "stm32f411xe" if SOC_STM32F411XE

config SYS_CLOCK_HW_CYCLES_PER_SEC
    default 100000000   # 100MHz

endif # SOC_STM32F411XE
```

**defconfig vs Kconfig.defconfig 的分工：**

| 文件 | 作用 | 用户能覆盖？ |
|------|------|-------------|
| `board_defconfig` | 默认使能的开关 | ❌ 只能改 prj.conf |
| `Kconfig.defconfig` | 默认值的设定 | ✅ 可以覆盖 |

## 6.6 board.cmake——烧录配置

```cmake
# boards/<arch>/<vendor>/board.cmake
# ★ 以下为示例，换成你的调试器参数
board_runner_args(stlink --stlink-cmd-opt="--connect-under-reset")    # ST-Link
board_runner_args(pyocd --target stm32f411xe)                         # pyOCD

include(${ZEPHYR_BASE}/boards/common/stlink.board.cmake)
include(${ZEPHYR_BASE}/boards/common/pyocd.board.cmake)
```

> 🔍 **如何确定调试器和 runner？**
> ```bash
> # 常用 runner：stlink（STM32）、jlink（通用）、pyocd（通用）、openocd（通用）、dfu-util（USB）
> west flash -h
> # 找跟你调试器一样的官方板子，抄它的 board.cmake
> ```

## 6.7 实战：最快的起步方法——复制并修改

### Step 1：找参考板

```bash
# 找跟你的 MCU 同型号的官方板子
find boards/<架构>/ -name "*.dts" | xargs grep -l "你的芯片型号"

# 举例：你用的是 STM32F411
find boards/arm/ -name "*.dts" | xargs grep -l "stm32f411"
# → nucleo_f411re, blackpill_f411ce 等
```

### Step 2：复制整个目录

```bash
cp -r boards/arm/nucleo_f411re boards/arm/myboard
cd boards/arm/myboard
mv nucleo_f411re.dts myboard.dts
mv nucleo_f411re.yaml myboard.yaml
mv nucleo_f411re_defconfig myboard_defconfig
```

### Step 3：修改板级 DTS

```devicetree
// myboard.dts 需要改的地方：

// 1. model 和 compatible
model = "My Custom Board";
compatible = "myvendor,myboard", "st,stm32f411";

// 2. chosen（看 PCB 上 UART 接哪个）
chosen {
    zephyr,console = &usart1;     // ← 根据你的原理图改
    zephyr,shell-uart = &usart1;
};

// 3. 改板级 LED/按键（没有就删掉）
// 4. 改 aliases

// 5. 外设配置——最核心！根据原理图改引脚
&usart1 {
    pinctrl-0 = <&usart1_tx_pa9 &usart1_rx_pa10>;
    pinctrl-names = "default";
    current-speed = <115200>;
    status = "okay";
};
```

### Step 4：改 YAML

```yaml
identifier: myboard
name: MyCustomBoard
arch: arm
ram_size: 131072
rom_size: 524288
supported:
  - stlink
```

### Step 5：编译验证

```bash
west build -b myboard samples/hello_world
# 成功 → 🎉 你的板子支持完成了！
# 报错 → 看错误信息调整
```

## 6.8 找答案——去哪查？

| 遇到问题 | 去哪找答案 |
|---------|-----------|
| 不知道 dts 怎么写引脚 | 搜官方板子 `board.dts` → grep 同类引脚 |
| 不知道 CONFIG_SOC_xxx 叫什么 | 搜 SoC 型号 → `arch/<arch>/soc/<vendor>/Kconfig*` |
| 不知道支持哪些 flash runner | `boards/common/*.board.cmake` |
| 编译报错 `board not found` | 确认 board yaml 的 identifier 名正确 |
| uart 不工作 | 查板子 UART 引脚和 pinctrl 配置 |
| 想知道某 SoC 系列有哪些板子 | `find boards/ -name "*.dts" | xargs grep -l "你的芯片"` |
| 想知道 dtsi 里定义了哪些节点 | `grep "^/ {" -A 200 zephyr/dts/<架构>/<厂商>/你的芯片.dtsi` |
| 板子启动就 panic | 检查 chosen 里的 console/uart 引脚对不对 |

## 6.9 把自定义板子放进应用目录（不修改 Zephyr 源码）

```bash
my_app/
├── src/
├── prj.conf
├── CMakeLists.txt
├── boards/                    ← 自定义板子放这里！
│   └── arm/
│       └── myboard/
│           ├── myboard.dts
│           ├── myboard.yaml
│           └── myboard_defconfig
└── dts/bindings/
```

Zephyr 构建系统会自动搜索应用目录下的 `boards/`。

```bash
cd my_app
west build -b myboard .
```

## 6.10 思维导图：一块板子的完整生命周期

```
┌──────────────────────────────────────────────┐
│              你的自定义板子                    │
├──────────────────────────────────────────────┤
│                                               │
│  1. board.yaml      → 告诉 west \"这个板子叫什么\" │
│  2. board.dts       → 告诉设备树 \"板上有什么\"   │
│     ├─ #include SoC dtsi （引入片上外设）       │
│     ├─ chosen          （指定系统资源）         │
│     ├─ leds/keys       （板级外设）             │
│     └─ aliases         （方便应用移植）         │
│  3. board_defconfig   → 告诉 Kconfig \"默认开什么\"│
│  4. Kconfig.defconfig → 告诉 Kconfig \"默认值多少\"│
│  5. board.cmake       → 告诉 west \"怎么烧录\"   │
│                                               │
│  用户编译：west build -b myboard .             │
│  → Zephyr 自动找到 boards/arm/myboard/        │
│  → 加载 dts + Kconfig + cmake                 │
└──────────────────────────────────────────────┘
```

---

# 第七章：实战案例——从零做一个外设驱动

用完整案例串联前几章讲的流程：**写 binding → 设备树 → prj.conf → 应用代码 → 调试验证。**

## 7.1 场景设定

- **板子：** NUCLEO-F411RE（STM32F411）
- **外设：** I2C 温湿度传感器 AHT20（假设 Zephyr 没有现成驱动）
- **需求：** 每秒读一次温湿度，通过串口打印

## 7.2 完整开发流程

### Step 1：确定芯片和 compatible

```bash
# 先查 Zephyr 有没有现成驱动
find zephyr/dts/bindings -name "*.yaml" | xargs grep -l "aht\|aosong"
# 没有 → 自己写

# 查 datasheet：厂商=aosong, 芯片=aht20, I2C 地址=0x38
# → compatible = "aosong,aht20"
```

### Step 2：写 Binding YAML

```yaml
# my_app/dts/bindings/aosong,aht20.yaml
compatible: "aosong,aht20"
include: [i2c-device.yaml]
description: Aosong AHT20 temperature and humidity sensor.

properties:
  alert-gpios:
    type: phandle-array
    required: false
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
    };
};
```

**验证：**
```bash
west build -b nucleo_f411re . -t devicetree
grep -A5 "aht20" build/zephyr/zephyr.dts
```

### Step 4：编辑 prj.conf

```
CONFIG_I2C=y
CONFIG_I2C_STM32=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y
CONFIG_PRINTK=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3
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
    int ret = i2c_write(i2c_dev, &reset, 1, AHT20_I2C_ADDR);
    if (ret) return ret;
    k_sleep(K_MSEC(20));

    uint8_t init[] = {AHT20_INIT_CMD, 0x08, 0x00};
    ret = i2c_write(i2c_dev, init, 3, AHT20_I2C_ADDR);
    if (ret) return ret;
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
    if (ret) return ret;
    k_sleep(K_MSEC(100));

    ret = i2c_read(i2c_dev, raw, 6, AHT20_I2C_ADDR);
    if (ret) return ret;

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
cd my_app
west build -b nucleo_f411re .
grep -A5 "aht" build/zephyr/zephyr.dts
grep "I2C" build/zephyr/.config
west flash
west build -t monitor
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
| 板级 YAML | `zephyr/boards/<厂商>/<板名>/*.yaml` |
| 驱动源码 | `zephyr/drivers/<类别>/` |
| 头文件（API） | `zephyr/include/zephyr/` |
| 设备树 API 全集 | `zephyr/include/zephyr/devicetree.h` |
| GPIO 标志位宏 | `zephyr/include/zephyr/dt-bindings/gpio/gpio.h` |
| Pinctrl 宏（芯片系列） | `zephyr/include/zephyr/dt-bindings/pinctrl/<芯片>.h` |
| Kconfig 源码定义 | `zephyr/**/Kconfig*`（grep 大法） |
| 示例代码 | `zephyr/samples/` |
| 官方文档（RST） | `zephyr/doc/` |
| 最终硬件描述 | `build/zephyr/zephyr.dts` |
| 最终配置 | `build/zephyr/.config` |
| 生成 DT 宏 | `build/zephyr/include/generated/devicetree_generated.h` |
| 烧录 runner 配置 | `boards/common/*.board.cmake` |

## B. 常用调试命令速查

```bash
# === 调试设备树 ===
west build -t devicetree          # 验证设备树正确性
cat build/zephyr/zephyr.dts       # 查看合并结果

# === 调试 Kconfig ===
west build -t menuconfig          # 交互式配置查看
cat build/zephyr/.config          # 最终配置

# === 调试构建 ===
west build -t pristine            # 清空重构建
west build -v                     # verbose 输出

# === 查看运行时 ===
west build -t monitor             # 串口监控
west flash                        # 烧录

# === 指定烧录 runner（按你的调试器选） ===
west flash --runner stlink        # ST-Link（STM32）
west flash --runner jlink         # J-Link（通用）
west flash --runner pyocd         # pyOCD（通用）
west flash --runner openocd       # OpenOCD（通用）
west flash --runner dfu-util      # DFU（通用）

# === 查看生成的宏 ===
cat build/zephyr/include/generated/devicetree_generated.h | head -50
```

## C. 常见问题排查清单

```
问题：设备没有工作
  ├─ 设备树 status = "okay"？
  ├─ CONFIG_xxx = y？
  ├─ device_is_ready() 检查了？
  ├─ pinctrl 引脚配对了？
  └─ 🎯 时钟配置是否正确？（设备树路线看 dts，Kconfig 路线看 .config）

问题：找不到 CONFIG 选项
  ├─ menuconfig 搜索（west build -t menuconfig，按 /）
  ├─ grep -r "config XXX" zephyr/ --include="Kconfig*"
  └─ 注意搜索 "config XXX" 不是 "CONFIG_XXX"

问题：设备树属性不生效
  ├─ cat build/zephyr/zephyr.dts 查看合并结果
  ├─ 确认 overlay 被正确加载了
  └─ 确认 compatible 字符串完全匹配

问题：API 不知道用哪个
  ├─ 找对应子系统的头文件
  ├─ 找对应子系统的 sample
  └─ 看 zephyr/doc/ 中的 RST 文档

问题：编译报错 "undefined DT_xxx"
  ├─ 确认设备树中有对应的节点
  ├─ 确认节点 status = "okay"
  └─ 确认宏名拼写正确（注意 - 变 _）

问题：west build -b myboard 找不到板子
  ├─ 确认 boards 目录在 Zephyr 源码或应用目录中
  └─ 确认 board yaml 的 identifier 字段正确

问题：烧录失败
  ├─ 确认 board.cmake 的 runner 配置正确
  ├─ 确认调试器连接正常
  └─ 检查 board.cmake include 路径
```

## D. 推荐学习路线

```
第 1 周：跑通官方 sample
  ├─ blinky（设备树 → prj.conf → main.c）
  ├─ hello_world（确认串口输出）
  └─ button（GPIO 输入和中断）

第 2 周：改设备树 + overlay
  ├─ 添加新 LED（改 overlay）
  ├─ 添加 I2C 传感器（用现有 binding）
  └─ 用 menuconfig 看配置

第 3 周：多线程应用
  ├─ 线程 + 消息队列
  ├─ work queue + 中断
  └─ Shell 子系统

第 4 周：自定义板子 + 驱动
  ├─ 基于官方板子 clone 自定义板子
  ├─ 写自己的 binding yaml
  └─ 调试板级支持
```

## E. 速记口诀

```
设备树管硬件，Kconfig管功能，C代码管逻辑。
属性名找binding，参数名搜Kconfig，API名去sample。
dts/chosen/alias → 设备获取三板斧。
找不到板子→查yaml identifier。
编译先用west build -t devicetree → 有问题早暴露。
```

---

> **最后记住：** Zephyr 的代码和文档本身是最好的学习资料。官方 sample 解决了 80% 的"第一步怎么做"的问题。剩下的 20% 是在具体报错时逐步调试查资料解决的。别怕读源码——源码才是最终真理。🦞
