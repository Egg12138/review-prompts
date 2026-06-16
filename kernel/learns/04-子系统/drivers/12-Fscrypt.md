<!-- Source: subsystem/fscrypt.md -->

# Fscrypt（文件系统加密）子系统

## 密钥管理（Key Management）

### 主密钥与派生密钥

Fscrypt 使用两级密钥体系：

- **Master Key（主密钥）**：存储在用户空间管理的 keyring（密钥环）中，每个加密目录可关联一个主密钥
- **Per-file Key（每文件密钥）**：由主密钥通过 KDF（Key Derivation Function，密钥派生函数）为每个文件派生，确保不同文件使用不同密钥

### 关键不变式

- **Key removal（密钥移除）使文件不可读**：一旦主密钥从 keyring 中移除，所有关联文件的数据和文件名均无法解密
- **Key zeroization（密钥清零）**：密钥使用完毕后必须通过 `fscrypt_zeroize_key()` 或类似机制将密钥内容从内存中清除，防止被冷启动攻击或内核内存转储泄露
- **Key identifier（密钥标识符）**：主密钥由加密哈希标识，存储在文件系统的加密上下文（encryption context）中，用于在 keyring 中查找对应密钥

### 常见错误

- 在文件已打开或 I/O 进行中时移除密钥，导致未处理的解密请求失败
- 多个目录共用一个主密钥，但移除其中一个目录密钥时未考虑其他目录的使用
- 未在错误路径上清零派生密钥缓冲区临时副本

---

## 加密上下文（Encryption Context）

### 存储位置

加密上下文（`struct fscrypt_context` / `struct fscrypt_context_v2`）存储以下位置之一：

| 存储位置 | 说明 |
|---------|------|
| **xattr（扩展属性）** | 存储在文件或目录的 `security.` 命名空间下的扩展属性中。最常见的方式 |
| **Superblock（超级块）** | 对于某些不可修改的文件系统（如 EROFS），上下文嵌入在超级块的元数据中 |

### 上下文内容

加密上下文包含以下关键字段：

- **Policy version（策略版本）**：`FSCRYPT_POLICY_V1` 或 `FSCRYPT_POLICY_V2`。V2 提供更强的密钥派生和更好的性能
- **Encryption modes（加密模式）**：用于内容加密的模式（如 `AES-256-XTS`）和用于文件名加密的模式（如 `AES-256-CBC`）
- **Master key identifier（主密钥标识符）**：用于在 keyring 中查找正确的主密钥
- **Nonce（随机数）**：每个目录唯一的随机值，用于密钥派生

### 策略继承与不变性

- **Children inherit（子文件继承父目录策略）**：在加密目录中创建的文件和子目录自动继承相同的加密策略
- **Cannot change after creation（创建后不可更改）**：一旦目录被标记为加密策略，其策略版本和加密模式不可更改。必须清空目录并重新创建才能更改策略

### 常见错误

- 在空目录上设置加密策略后，未验证目录是否确实为空（存在已有文件将导致 `EEXIST`）
- 尝试修改已存在的加密目录的策略版本

---

## IV（Initialization Vector，初始化向量）要求

### IV 唯一性

- IV **必须**对每个文件和每个逻辑块（logical block）唯一
- IV 重用（IV reuse）会破坏机密性（confidentiality），使密文易受统计分析攻击
- 每个文件使用独立的密钥派生防止不同文件的 IV 冲突

### 不同模式的 IV 方案

| 加密模式 | IV 生成方式 | 说明 |
|---------|------------|------|
| AES-256-XTS | 基于逻辑块号 | 每个 512 字节或 4096 字节块使用递增的 IV |
| AES-256-CBC-CTS | 基于逻辑块号 + 文件非重复值 | 文件名加密的 IV 包含文件的 nonce 和块偏移 |
| AES-256-ESSIV | 基于加密密钥的哈希 | 为 CBC 模式提供每块独特的 IV（传统方案） |

### 常见错误

- 未正确处理文件截断（truncation）后的逻辑块号，导致扩展文件后 IV 重用
- 对于内联加密硬件，未验证硬件的 IV 生成方式与软件期望一致

---

## 策略执行（Policy Enforcement）

### 目录级加密

- 加密策略设置在目录上，策略会传播到该目录中的所有文件和子目录
- 允许在空目录上设置策略（`FS_IOC_SET_ENCRYPTION_POLICY` ioctl）
- 非空目录设置策略将返回错误

### 文件级加密

- 文件单独继承其父目录的加密策略
- 文件加密采用**内容加密**（content encryption，数据加密）和**文件名加密**（filenames encryption，目录项加密）

### 策略验证流程

1. 用户空间调用 `FS_IOC_SET_ENCRYPTION_POLICY` 设置策略
2. 内核验证目录是否为空
3. 内核生成 encryption context 写入 xattr
4. 所有新创建的文件和子目录自动继承该策略

### 常见错误

- 在文件系统 mount 时未正确设置 `test_dummy_encryption` 或 `inlinecrypt` 选项
- 对非空目录设置策略（操作将失败，但用户空间可能忽略错误码）

---

## Invariants

- 主密钥移除后，关联文件不可读（数据和文件名均无法解密）
- 密钥使用后必须在内存中清零（zeroized）
- 加密策略一旦设置后不可更改
- 新文件自动继承父目录的加密策略
- IV 必须对每个文件和每个逻辑块唯一
- 加密上下文必须在数据写入前设置完成
- 加密数据的填充（padding）必须对齐到块大小（通常为 16 字节）

## Failure Modes

| 违反场景 | 结果 |
|---------|------|
| 移除正在使用的密钥 | 未处理的解密请求失败（I/O 错误） |
| 密钥未清零 | 内核内存转储泄漏密钥信息，冷启动攻击风险 |
| 尝试更改已有策略 | `EEXIST` 错误返回 |
| IV 重用（如文件截断后） | 机密性被破坏 |
| 未对齐的加密数据块 | 加密引擎硬件错误或性能下降 |
| 明文文件名出现在加密目录 | 敏感文件名泄露（如 `/home/user/encrypted/tax-2025.pdf`） |
| 在数据操作之后才设置加密上下文 | 明文数据写入磁盘后无法加密 |

## Quick Checks

- **Key availability（密钥可用性）**：在解密操作前确认主密钥在 keyring 中可用
- **Encryption context timing**：加密上下文是否在数据写入操作之前设置完成？
- **Padding correctness**：加密数据是否已正确填充到块大小（16 字节对齐）？
- **Filenames protection**：加密目录中是否存在明文文件名？
- **IV uniqueness**：文件截断（truncation）后，文件扩展时的逻辑块号是否会产生 IV 冲突？
- **Policy inheritance**：在加密目录中创建新文件时，是否验证了策略自动继承？
- **Key zeroization paths**：在所有错误路径上是否已清零派生密钥的临时副本？
