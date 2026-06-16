<!-- Source: subsystem/usb-storage.md -->

# USB 存储子系统详解

## unusual_devs.h 条目约定（Entry Conventions）

在 `UNUSUAL_DEV()` 条目中指定不必要的子类（subclass）或协议（protocol）覆盖，会导致驱动在每次设备插入时发出 `dev_notice`，要求用户报告不需要的条目（参见 `get_device_info()` 在 `drivers/usb/storage/usb.c`）。这会给最终用户造成持续的日志噪音。

`UNUSUAL_DEV()` 宏定义在 `drivers/usb/storage/unusual_devs.h`，接受十个位置参数。子类和协议位置分别是第七和第八个参数：

```c
UNUSUAL_DEV(idVendor, idProduct, bcdDeviceMin, bcdDeviceMax,
            vendor_name, product_name,
            use_protocol,   /* 子类：USB_SC_* 值 */
            use_transport,  /* 协议：USB_PR_* 值 */
            init_function, Flags)
```

注意：内核的字段名历史上容易混淆。`struct us_unusual_dev`（`drivers/usb/storage/usb.h`）中的 `useProtocol` 字段持有 `USB_SC_*` 子类代码，而 `useTransport` 持有 `USB_PR_*` 协议代码。

### 参数语义

| 值 | 含义 |
|----|------|
| `USB_SC_DEVICE`（0xff） | 使用设备自身报告的 `bInterfaceSubClass` |
| `USB_PR_DEVICE`（0xff） | 使用设备自身报告的 `bInterfaceProtocol` |
| 特定的 `USB_SC_*` 值 | 覆盖设备的子类（例如 `USB_SC_SCSI` = 0x06） |
| 特定的 `USB_PR_*` 值 | 覆盖设备的协议（例如 `USB_PR_BULK` = 0x50） |

当显式覆盖与设备已报告的值一致时，`get_device_info()` 检测到冗余并记录通知——除非条目携带 `US_FL_NEED_OVERRIDE` 标志（定义在 `include/linux/usb_usual.h`），该标志会抑制对有意需要覆盖的条目的警告。

### 条目统计

`unusual_devs.h` 中大约 85% 的 `UNUSUAL_DEV()` 条目使用 `USB_SC_DEVICE, USB_PR_DEVICE`。带有显式覆盖的条目是少数，通常是因为设备错误报告其子类或协议，或者条目需要非标准的传输处理器。

### 交叉引用设备描述符

来自 `/sys/kernel/debug/usb/devices` 的设备描述符输出显示设备自身报告的接口值：

```
I:* If#= 0 Alt= 0 #EPs= 2 Cls=08(stor.) Sub=06 Prot=50 Driver=usb-storage
```

这里 `Sub=06` 对应 `USB_SC_SCSI`，`Prot=50` 对应 `USB_PR_BULK`。如果 `UNUSUAL_DEV()` 条目显式指定这些值而非使用 `USB_SC_DEVICE` / `USB_PR_DEVICE`，则覆盖是冗余的，驱动会发出不必要的覆盖通知。

```c
// 正确：让驱动使用设备自身报告的值
UNUSUAL_DEV(0x1234, 0x5678, 0x0100, 0x0100,
    "Vendor", "Product",
    USB_SC_DEVICE, USB_PR_DEVICE, NULL,
    US_FL_NO_ATA_1X)

// 错误：设备已报告 Sub=06 Prot=50 时不必要的覆盖
UNUSUAL_DEV(0x1234, 0x5678, 0x0100, 0x0100,
    "Vendor", "Product",
    USB_SC_SCSI, USB_PR_BULK, NULL,
    US_FL_NO_ATA_1X)
```

## Invariants（不变规则）

- 除非确实需要覆盖设备的自报告值，否则第七、第八参数应使用 `USB_SC_DEVICE` / `USB_PR_DEVICE`（0xff）。
- 带有显式 `USB_SC_*` / `USB_PR_*` 值的新 `UNUSUAL_DEV()` 条目，其提交信息必须解释覆盖的原因。
- `US_FL_NEED_OVERRIDE` 标志可抑制不必要的覆盖警告，但仅应在条目明确需要强制覆盖时使用。

## Failure Modes（失败模式）

- 指定与设备自报告值相同的显式子类/协议覆盖 → 每次设备插入时产生 `dev_notice` 日志噪音。
- 提交信息未解释显式覆盖的原因 → 审阅者无法判断覆盖是否必要。

## Quick Checks（快速检查要点）

- 当补丁添加带有显式 `USB_SC_*` / `USB_PR_*` 值的 `UNUSUAL_DEV()` 条目（而非 `USB_SC_DEVICE` / `USB_PR_DEVICE`）时，提交信息应解释为什么需要覆盖。
- 如果提交包含设备描述符输出，将 `Sub=` 和 `Prot=` 字段与条目的第七、第八参数进行比较，以检测冗余覆盖。
