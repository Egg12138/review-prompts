<!-- Source: subsystem/syscall.md -->

# 系统调用 ABI 核心知识

## ABI 兼容性

### Invariants

- 向 syscall 添加额外参数不会破坏 ABI，**前提是不更改现有参数**
- 这意味着新的参数必须追加在现有参数之后，而不是插入或修改已有参数

---

## Syscall 参数信任边界

### 核心概念

Syscall 参数来自用户控制的寄存器或栈槽。仅当设置了特定标志时才具有意义的参数可能在该标志缺失时包含任意垃圾值 —— **用户空间不需要清零未使用的参数**。

### 关键不变性

当 syscall 参数通过 `copy_from_user()`/`copy_struct_from_user()` 复制到内核 struct 时，每个字段继承其源参数的信任边界。即使字段在 C 代码中看起来已初始化，在标志守卫之外它仍然是垃圾。

```c
// 示例：假设 sysctl 参数在 flag 门控之外不安全
long sys_mycall(int cmd, unsigned long arg)
{
    struct my_param __user *p = (void __user *)arg;

    if (cmd == MY_CMD_SET) {
        // 此处 arg 作为指向用户结构的指针，是可信任的
        struct my_param kp;
        if (copy_from_user(&kp, p, sizeof(kp)))
            return -EFAULT;
        // 使用 kp 中的字段
    }
    // 此处 arg 的内容未经 cmd==MY_CMD_SET 验证就是不可信的
}
```

### Failure Modes

当重构将检查跨越标志守卫移动时，**必须验证检查使用的每个变量在更广泛的作用域中有效**。在标志守卫之外使用标志门控的 syscall 参数进行验证、算术或比较可能导致：
- 处理完全由用户控制的垃圾值
- 有效性检查被绕过
- 信息泄露

**REPORT as bugs**：任何在标志守卫（flag gate）作用域之外使用标志门控 syscall 参数进行的验证、算术或比较。

---

## Quick Checks

- syscall 参数只有在经过验证后才是可信任的
- 标志守卫之外的 syscall 参数字段可能包含任意用户态控制的值
- 重构时，验证每个移动的检查在其新位置中使用的变量是否都有效
