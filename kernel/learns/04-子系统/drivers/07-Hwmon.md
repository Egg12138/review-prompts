<!-- Source: (general kernel knowledge) -->

# Hwmon（硬件监控）子系统

## 概述

Hwmon（Hardware Monitoring，硬件监控）子系统是 Linux 内核中用于监控硬件传感器（温度、电压、风扇转速、电流、功率等）的框架。它为传感器设备提供统一的 sysfs 接口，使用户空间工具（如 `lm-sensors`）能方便地读取传感器数据。

---

## 设备注册与 API

### 关键数据结构

Hwmon 设备通过 `struct hwmon_channel_info` 和 `struct hwmon_chip_info` 定义传感器通道和芯片属性。

```c
// 定义温度传感器通道
static const struct hwmon_channel_info *foo_info[] = {
    HWMON_CHANNEL_INFO(temp,
        HWMON_T_INPUT | HWMON_T_MAX | HWMON_T_CRIT,
        HWMON_T_INPUT),
    NULL
};

// 定义芯片信息
static const struct hwmon_chip_info foo_chip_info = {
    .ops = &foo_hwmon_ops,
    .info = foo_info,
};
```

### 注册函数

| 函数 | 说明 |
|------|------|
| `devm_hwmon_device_register_with_info(dev, name, chip, info, groups)` | 推荐的注册方式，带芯片信息 |
| `hwmon_device_register_with_groups(dev, name, data, groups)` | 旧的注册方式，使用属性组 |
| `devm_hwmon_device_register_with_groups()` | 上述的 devm 版本 |

### 不变式

- 使用 `devm_` 版本自动处理设备注销，避免资源泄漏
- `struct hwmon_ops` 中的回调函数（`is_visible`、`read`、`write`）在调用者上下文中执行，可能持有锁，因此回调中禁止可能导致休眠的长时间操作
- 注册后设备立即可见，所有数据结构和回调必须在注册前完全初始化

---

## 属性可见性与 is_visible

### 类似 sysfs 的条件可见性

Hwmon 通过 `is_visible` 回调控制每个传感器通道的 sysfs 属性是否可见：

```c
static umode_t foo_hwmon_is_visible(const void *data,
                                     enum hwmon_sensor_types type,
                                     u32 attr, int channel)
{
    const struct foo_data *foo_data = data;

    if (type == hwmon_temp && !foo_data->has_temp_sensor)
        return 0;  // 该属性不可见

    return 0444;  // 只读
}
```

### 不变式

- 返回 0 表示该属性**不创建**，调用者不能通过 sysfs 访问
- 返回 `SYSFS_GROUP_INVISIBLE` 可隐藏整个通道组
- `is_visible` 在注册时调用一次，运行时不可变（除非注销后重新注册）

---

## 读写回调实现

### read 回调

`read` 回调根据类型返回传感器值。返回值是错误码，实际值通过参数指针返回。

```c
static int foo_hwmon_read(struct device *dev,
                           enum hwmon_sensor_types type,
                           u32 attr, int channel, long *val)
{
    struct foo_data *data = dev_get_drvdata(dev);

    switch (type) {
    case hwmon_temp:
        switch (attr) {
        case hwmon_temp_input:
            *val = foo_read_temperature(data, channel);
            return 0;
        default:
            return -EOPNOTSUPP;
        }
    default:
        return -EOPNOTSUPP;
    }
}
```

### 不变式

- 注册成功前不应调用 `read`/`write` 回调
- 回调必须处理所有在 `HWMON_*_INPUT` 等标志中声明的属性
- 对于不支持的属性，返回 `-EOPNOTSUPP`
- 在回调中访问共享数据时需要考虑并发保护（通常使用 mutex）

---

## 故障模式

### 1. 传感器读取失败处理不当

传感器硬件可能暂时不可用（如 I2C 通信错误），此时应返回负 errno 而非错误数据。

```c
// 错误：忽略硬件错误，返回未初始化或错误的值
static int foo_hwmon_read(struct device *dev, ..., long *val)
{
    *val = foo_read_register_silent(data);
    return 0;  // 即使读取失败也返回 0
}

// 正确：将底层错误传播给调用者
static int foo_hwmon_read(struct device *dev, ..., long *val)
{
    int ret = foo_read_register(data, val);
    if (ret < 0)
        return ret;
    return 0;
}
```

### 2. 未处理转换时间

某些传感器在模式切换或通道选择后需要时间完成转换。在转换完成前读取可能返回旧数据或 0。

### 3. 精度损失

从固件或硬件寄存器读取的值可能有固定小数位。驱动应正确使用 `val` 的单位约定（如 `temp` 以毫摄氏度为单位，`fan` 以 RPM 为单位），避免精度损失。

---

## 快速检查清单

- **注册顺序**：在 `hwmon_device_register_with_info()` 前确保私有数据已初始化并可通过 `dev_get_drvdata()` 获取
- **错误传播**：`read` 回调中底层硬件错误应传播，而非静默返回 0
- **并发保护**：`read`/`write` 回调可能并发调用，共享数据需适当的锁保护
- **属性声明**：`hwmon_channel_info` 中的标志必须与 `read`/`write` 回调中实际处理的属性匹配
- **devm 使用**：优先使用 `devm_hwmon_device_register_with_info()` 自动管理生命周期
