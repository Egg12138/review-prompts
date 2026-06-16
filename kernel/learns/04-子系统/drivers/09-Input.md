<!-- Source: subsystem/input.md -->

# Input（输入）子系统

## 概述

Linux Input 子系统处理键盘、鼠标、触摸屏、游戏手柄等输入设备的事件报告。本文覆盖设备生命周期、注册序列、事件处理、Force-Feedback 内存管理以及 devm 资源集成等关键不变式和故障模式。

---

## 设备生命周期与注册

### 分配与注册序列

#### 分配

输入设备必须使用 `input_allocate_device()` 或 `devm_input_allocate_device()` 分配。

#### 注册后的归属转移

一旦 `input_register_device()` 成功，input 核心接管部分生命周期管理。

```c
// 安全模式：某些驱动使用统一失败路径
input_unregister_device(dev);  // 注销注册的设备
dev = NULL;                    // 防止后续 free
input_free_device(dev);        // input_free_device(NULL) 是空操作
```

**安全模式**：某些驱动使用统一失败路径——先调用 `input_unregister_device(dev)`，然后设置 `dev = NULL`，再调用共享的 `input_free_device(dev)`。这种"防御性 NULL"模式是安全的，因为 `input_free_device(NULL)` 是空操作。

**故障模式**：在持有成功注册的设备的指针上调用 `input_free_device()`。但上述"防御性 NULL"模式安全，不应报告为 bug。

### 注册前后行为

| 状态 | 调用 `input_event()` | 结果 |
|------|---------------------|------|
| 分配后、注册前 | **可以安全调用** | 更新内部状态（如 `keybit`、绝对值轴的当前值），但不会向 handler 传播事件 |
| 注册后 | **正常处理** | 事件传播给用户空间 handler |

注册前可调用 `input_event()` 的设计允许驱动在设备对用户空间可见之前请求 IRQ 并处理初始状态同步。

### 能力（Capabilities）

- input 核心自动向所有输入设备添加 `EV_SYN`/`SYN_REPORT` 能力。手动添加是多余的。
- `input_register_device()` 是设备对用户空间可见的时点。所有驱动私有数据和回调（包括 `input_set_drvdata()`）**必须**在调用注册之前完全初始化。注册后，核心或用户空间 ioctl 可能立即调用回调。

### 生命周期保证

input 核心使用引用计数管理 `input_dev` 对象。即使在 `input_unregister_device()` 被调用后，设备内存将保持有效直到最后一个用户空间引用（如通过 evdev 打开的）被释放。

---

## 回调时序与序列化

### 不变式

一旦 `input_register_device()` 被调用，驱动必须完全初始化并为 `input_dev->open()` 和 `input_dev->close()` 做好准备。如果用户空间 handler 已经在等待，核心可能在注册函数返回前就调用 `open()`。

### 序列化

- input 核心使用 `input_dev->mutex` 序列化 `open()` 和 `close()`
- 如果驱动需要将其他方法（如自定义 `ioctl` 或 `setkeycode`）与设备的打开/关闭状态同步，必须显式获取 `input_dev->mutex`
- 要检查设备是否活跃（有用户且未被抑制），使用 `input_device_enabled()`，但**必须**在持有 `input_dev->mutex` 时调用

### 拆卸

`input_unregister_device()` 在设备当前为打开状态时会自动调用驱动的 `close()` 方法。

### 故障模式

假设回调不会立即被调用，或对回调时序做出不正确假设，导致未初始化的数据结构被访问。

---

## Force-Feedback 内存管理

### 特殊所有权语义

`input_ff_create_memless()` 对私有数据有特殊的生命周期语义：

```c
// 正确：data 由 kmalloc 分配，所有权传递给 input 核心
struct foo_data *data = kmalloc(sizeof(*data), GFP_KERNEL);
if (!data)
    return -ENOMEM;
ret = input_ff_create_memless(dev, data, play_effect);
if (ret)
    kfree(data);  // 失败时需要手动释放
```

### 不变式

- `input_ff_create_memless(dev, data, play_effect)` **接管** `data` 指针的所有权
- 当 input 设备销毁时，该数据将通过 `kfree()` 自动释放
- data **必须**通过 `kmalloc()`（或类似）分配，**不得**由 devm 管理

```c
// 错误：使用 devm 分配或手动释放数据
data = devm_kzalloc(dev, sizeof(*data), GFP_KERNEL);
input_ff_create_memless(dev, data, play_effect);
// 设备销毁时 kfree(data) 是 double-free，因为 devm 也会释放
```

### 故障模式

手动释放传递给 `input_ff_create_memless()` 的数据，或使用 `devm_kzalloc()` 分配——这将导致 double-free 或释放已管理内存。

---

## Managed Resources（devm）集成

### 关键知识点

| 项目 | 说明 |
|------|------|
| 注册包装器 | 没有 `devm_input_register_device()`。通过 `devm_input_allocate_device()` 分配的输入设备应使用标准 `input_register_device()` 注册 |
| 自动注销 | input 核心能识别 managed 设备，并自动设置一个回调在 provider 设备解除绑定时调用 `input_unregister_device()` |
| 冗余父赋值 | 使用 `devm_input_allocate_device(dev)` 时，核心自动设置 `input_dev->dev.parent = dev`，手动赋值是多余的 |

### 故障模式

- 对通过 `devm_input_allocate_device()` 分配的设备显式调用 `input_unregister_device()`——通常冗余，可能导致双重注销
- 混合使用 managed 和普通资源时顺序违反生命周期约束

### 资源顺序建议

驱动应避免混合使用 managed 和普通资源。如果两者都使用，非 managed 资源（如手动 IRQ 请求）应在 managed 资源**之后**获取，并在 managed 清理触发**之前**手动释放。

---

## 事件报告与同步

### 不变式

每一组逻辑事件（如坐标对或一组按键变化）后必须调用 `input_sync()`。没有同步，用户空间 handler 可能无法收到更新后的状态。

```c
// 正确：每组事件后调用 input_sync()
input_report_key(dev, BTN_TOUCH, 1);
input_report_abs(dev, ABS_X, x);
input_report_abs(dev, ABS_Y, y);
input_sync(dev);
```

### 建议

- 使用 `input_report_key()`、`input_report_abs()`、`input_report_rel()` 等专用报告辅助函数，而非通用的 `input_event()`
- input 核心会过滤掉冗余事件（两次报告相同的值），但驱动仍应避免不必要的报告以减少事件投递路径的开销
- 锁：`input_event()` 和所有报告辅助函数获取 `input_dev->event_lock`（一个 spinlock）并本地禁用中断。驱动调用这些函数时必须确保不持有可能导致死锁的锁

### 故障模式

事件报告路径（ISR 或工作线程）末尾缺少 `input_sync()` 调用，导致用户空间看到不一致的设备状态。

---

## 维护者风格偏好

### 代码风格

- **注释**：优先使用 C 风格注释 `/* ... */` 而非 C++ 风格的 `// ...`
- **能力设置**：建议使用 `input_set_capability(dev, type, code)` 而非直接的位操作（如 `__set_bit(code, dev->evbit)`）。直接的位操作在循环或重复性构造中可以接受
- **清理机制**：在新代码或大规模重构中使用 `guard(mutex)(&input_dev->mutex)`。鼓励使用 `__free()` 管理局部资源，如 `u8 *buf __free(kfree) = ...`
- **错误命名**：使用 `error` 或 `err` 作为仅保存负错误码和 0 的变量名。使用这些变量的函数应在成功路径显式 `return 0;`
- **显式失败路径**：不要在有多个失败点的函数中使用 `return action(...);` 模式。使用展开形式：

```c
error = action(...);
if (error) {
    /* 需要时报告错误 */
    return error;
}
return 0;
```

---

## 快速检查清单

- **标识与初始化**：确保 `input_dev->name` 已赋值。验证所有私有结构体字段和 `input_set_drvdata()` 在 `input_register_device()` **之前**初始化
- **层次结构**：验证 `input_dev->dev.parent` 已设置。如果驱动使用 `devm_input_allocate_device()`，父设备自动设置，手动赋值应删除（冗余）
- **中断时序**：虽然注册前报告事件是安全的，但驱动必须确保硬件完全就绪（上电、时钟使能、寄存器可访问）后才允许中断处理函数运行
- **Mutex 保护**：如果驱动调用 `input_device_enabled()`，验证其持有 `input_dev->mutex`
- **事件同步**：验证事件报告路径（ISR 或工作线程）以 `input_sync()` 调用结束
- **FF 内存**：如果使用 `input_ff_create_memless()`，验证私有数据**不是** devm 分配的且**没有**手动释放
