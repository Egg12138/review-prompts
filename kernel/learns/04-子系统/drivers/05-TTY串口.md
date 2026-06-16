<!-- Source: subsystem/tty.md -->

# TTY/串口（Serial）子系统

## 概述

Linux TTY 和串口子系统负责终端设备和串行通信。本文聚焦 UART 设备注册与回调时序、8250 驱动模块架构等关键不变式和故障模式。

---

## UART 设备注册与回调时序

### 不变式

在运行时 PM 启用之前，从回调中调用运行时 PM API 会导致设备 probe 期间的循环依赖，造成任务阻塞和工作线程挂起——probe 函数无限期阻塞等待无法完成的运行时 PM 操作。

### 原因

`uart_add_one_port()` 可能在注册期间**同步调用**驱动回调，包括通过 `uart_change_pm()` 调用 `uart_ops->pm()` 回调。如果这些回调调用了 `pm_runtime_resume_and_get()` 或 `pm_runtime_put_sync()` 而运行时 PM 尚未在该设备上启用，运行时 PM 核心会尝试对未初始化的设备进行操作，导致循环等待条件。

### 正确顺序

```c
// 错误：端口注册后才启用运行时 PM
ret = uart_add_one_port(drv, uport);    // 可能调用 ops->pm()
if (ret)
    return ret;
devm_pm_runtime_enable(dev);            // 太晚了——回调已调用

// 正确：端口注册前先启用运行时 PM
devm_pm_runtime_enable(dev);            // 先启用
ret = uart_add_one_port(drv, uport);    // 现在 ops->pm() 使用运行时 PM 是安全的
if (ret)
    return ret;
```

### `uart_add_one_port()` 期间调用的回调

这些回调通过以下调用链从 `uart_add_one_port()` 到达（`drivers/tty/serial/serial_core.c`）：
`uart_add_one_port()` -> `serial_ctrl_register_port()` -> `serial_core_register_port()` -> `serial_core_add_one_port()` -> `uart_configure_port()`

- `uart_ops->config_port()`：当 `port->flags` 中设置了 `UPF_BOOT_AUTOCONF` 时调用
- `uart_ops->pm()`：通过 `uart_change_pm()` 给端口上电，对于非控制台端口还会再断电
- `uart_ops->set_mctrl()`：上电后去激活调制解调器控制线
- `port->rs485_config()`：通过 `uart_rs485_config()` 在设置 `SER_RS485_ENABLED` 时调用

### 故障模式

将串口驱动转换为使用运行时 PM 时，如果驱动的 `uart_ops->pm` 回调调用任何运行时 PM API，则必须在 `uart_add_one_port()` **之前**调用 `pm_runtime_enable()` 或 `devm_pm_runtime_enable()`。

---

## 8250 模块架构

### 关键知识点

在 8250 串口驱动的文件之间移动代码而不理解模块结构，会导致链接时出现未定义符号错误或破坏 depmod 的循环模块依赖。这些错误仅在 `CONFIG_SERIAL_8250=m`（模块化配置）时显现。

### 模块划分

8250 驱动拆分为两个内核模块，具有**单向依赖**关系。定义在 `drivers/tty/serial/8250/Makefile`：

| 模块 | 目标文件 | 角色 |
|------|---------|------|
| `8250_base.ko` | `8250_port.o`（始终）；`8250_dma.o`、`8250_dwlib.o`、`8250_fintek.o`、`8250_pcilib.o`、`8250_rsa.o`（条件编译） | 共享基础功能 |
| `8250.ko` | `8250_core.o`、`8250_platform.o`（始终）；`8250_pnp.o`（条件编译） | 主驱动 |

### 依赖方向

`8250.ko` **依赖** `8250_base.ko`（单向）。`8250_port.c` 导出符号供 `8250_core.c` 使用。`8250_base.ko` 中的代码**不得**调用 `8250.ko` 中的符号，否则会创建循环依赖。

### 审查要点

当审查在 `drivers/tty/serial/8250/` 中的 `.c` 文件间移动函数的补丁时：

1. 查阅 `Makefile` 确定哪些 `.o` 文件属于哪个 `.ko` 模块（查看 `8250-y +=` 与 `8250_base-y +=` 赋值）
2. 如果函数从一个模块移到另一个模块但仍被原始模块中的代码调用，必须用 `EXPORT_SYMBOL()` 或 `EXPORT_SYMBOL_GPL()` 导出
3. 如果函数从 `8250_base.ko` 移到 `8250.ko` 但仍被 `8250_base.ko` 调用，这引入了违反单向设计的反向依赖

### 逆向依赖问题的架构解决方案

- 将代码移回 `8250_base.ko` 以保持单向依赖
- 使用函数指针间接调用：在初始化时传递回调而非直接符号引用
- 在 `8250_base.ko` 中创建注册函数，存储供以后使用的指针

### 故障模式

在 `drivers/tty/serial/8250/` 中的文件之间移动代码时，未检查 `Makefile` 就跨模块边界移动函数，破坏了单向依赖设计。

---

## 快速检查清单

- **`uart_ops` 回调中的运行时 PM**：如果任何回调使用运行时 PM API，验证 `pm_runtime_enable()` 在 `uart_add_one_port()` **之前**调用
- **注册期间的回调调用**：`uart_add_one_port()` 可能在端口注册期间同步调用 `pm()`、`config_port()`、`set_mctrl()` 和 `rs485_config()`
- **8250 跨模块代码移动**：当代码在 `drivers/tty/serial/8250/` 的文件间移动时，检查 `Makefile` 以验证模块边界没有被违反
