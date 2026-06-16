<!-- Source: subsystem/nfsd.md -->

# NFS 服务器（NFSD）子系统

NFSD（`fs/nfsd/`）实现 Linux NFS 服务器（v2/v3/v4.x）。关键子系统：XDR 编解码器、stateid/delegation 状态机、文件句柄验证、客户端生命周期、回调、会话槽位。NFSv4 XDR 有自动生成的代码（xdrgen）。

## 文件布局

| 文件 | 领域 |
|------|------|
| `nfs4xdr.c`、`nfs3xdr.c` | XDR 编解码、页面编码 |
| `nfs4state.c`、`nfs4proc.c` | NFSv4 状态、操作、拷贝卸载 |
| `nfs3proc.c`、`nfsproc.c` | NFSv2/v3 操作 |
| `vfs.c`、`nfsfh.c`、`nfsfh.h` | VFS 接口、文件句柄、splice |
| `nfs4callback.c` | 回调 |
| `nfs4layouts.c` | pNFS 布局 |
| `filecache.c`、`nfscache.c` | 文件缓存、DRC |
| `export.c`、`nfsctl.c`、`netlink.c` | 导出管理、管理、netlink |
| `nfs4recover.c` | 宽限期、回收 |
| `state.h`、`netns.h` | 数据结构 |

## 信任边界

```
XDR 解码（未受信任）→ fh_verify() → nfs4_preprocess_stateid_op() → VFS（受信任）
```

所有客户端提供的数据在验证前都是未受信任的。`fh_verify()` 验证文件句柄和权限。`nfs4_preprocess_stateid_op()` 为有状态操作验证 stateid。

## XDR 编解码器

### 解码验证要求

`xdr_stream_decode_*()` 函数返回错误码，使用解码值前必须检查。解码后的长度和计数是客户端控制的，在用作分配大小、循环边界或数组索引前需要边界检查。

- 检查返回值后再使用解码变量
- 在 `kmalloc()` 之前进行边界检查——`count * sizeof(...)` 计算使用 `check_mul_overflow()`
- 验证数组索引（例如 slot 索引对 `maxreqs`、opnums 的检查）

### 编码要求

- 使用前检查 `xdr_reserve_space()` 返回值是否为 NULL
- 检查 `xdr_stream_encode_*()` 返回值
- 只在编码成功后才完成状态变更——如果编码在不可逆变更（close、revoke、rename）后失败，操作不能安全重试
- 在 `nfs4_put_stid()` 之前复制 stateid，而不是之后（put 可能释放）
- 从客户端提供的 `maxcount` 中减去头部开销
- 在编码循环中为尾部字段（eof、count）预留空间
- 无条件编码所有 RFC 要求的字段

### 重放缓存

`so_replay` / `rp_buf` 必须在编码成功后填充。编码失败后无条件 `memcpy` 到 `rp_buf` 会缓存垃圾数据。

### 受信任的来源

xdrgen 代码（`nfs4xdr_gen.c`）有内置验证。`fh_verify()` 之后 `fh_dentry` 的元数据是受信任的。

## 引用计数

NFSD 使用多种具有不同语义的引用计数：

| 计数器 | 防止 | 用途 |
|--------|------|------|
| `sc_count` | 释放 | Stateid 生命周期 |
| `cl_nfsdfs.cl_ref` | 释放 | nfsdfs 客户端对象生命周期 |
| `cl_rpc_users` | 取消哈希 | 使客户端在传入 RPC 复合操作期间保持活跃 |
| `cl_cb_inflight` | 客户端销毁 | 跟踪在传出的回调 |

**`cl_rpc_users` vs `cl_nfsdfs.cl_ref` vs `cl_cb_inflight`：**

`cl_nfsdfs.cl_ref` 仅防止 nfsdfs 客户端对象被释放。`cl_rpc_users` 防止取消哈希——用于传入 RPC 复合操作和需要客户端在复合操作生命周期后保持活跃的异步操作（如 copy worker）。`cl_cb_inflight` 跟踪传出的回调；`nfsd4_run_cb()` 内部递增它，`destroy_client()` 通过 `nfsd4_shutdown_callback()` 等待它耗尽。不要混淆 `cl_rpc_users` 和 `cl_cb_inflight`——它们保护不同方向的通信。

**分配时机**：仅在验证完成后才将资源分配给结构体字段。在验证通过前使用临时变量。来自 `nfsd_set_fh_dentry()` 修复的模式（b3da9b141578）。

**引用计数配对：**
- `nfs4_get_stid()` / `nfs4_put_stid()`
- `nfsd_file_get()` / `nfsd_file_put()`
- `exp_get()` / `exp_put()`
- `fh_put()`（复制语义）
- `nfsd_net_try_get()` / `nfsd_net_put()`
- 异步拷贝：`cl_rpc_users` / `put_client_renew()`
- `nfs4_get_stateowner()` / `nfs4_put_stateowner()`

转移语义（函数"窃取"引用）在有文档说明的情况下是可接受的。

**Stateowner 引用计数（`so_count`）：** 哈希表/链表成员资格（`cl_ownerstr_hashtbl` 和 `cl_openowners`）不是计数的引用——`hash_openowner()` 不做 `atomic_inc`。`alloc_stateowner()` 设置的 `so_count=1` 是转移给调用者的创建引用。Stateid 通过 `init_open_stateid()` -> `nfs4_get_stateowner()` 获取额外的计数引用（存储在 `stp->st_stateowner` 中）。通过 `find_openstateowner_str()` / `find_lockowner_str_locked()` 的查找会为调用者增加 `so_count`。

`release_openowner()` 恰好消耗一个调用者提供的引用：它取消哈希、排空 stateid（每个 `nfs4_free_ol_stateid` 通过 `nfs4_put_stateowner` 丢弃其所有者引用），然后执行一次最终的 `nfs4_put_stateowner`，使 `so_count=0` 并释放所有者。两个调用者（`find_or_alloc_open_stateowner` 和 `__destroy_client`）持有一个 pin（`find_openstateowner_str` 引用或显式 `nfs4_get_stateowner`），由 `release_openowner` 消耗——不要在 `release_openowner()` 返回后再添加额外的 `nfs4_put_stateowner()`。

## 文件句柄生命周期

`fh_dentry`、`fh_export` 和 `d_inode()` 在 `fh_verify()` 成功前为 NULL/无效。访问在验证前会导致 NULL 解引用或使用过期数据。

**权限标志：** `NFSD_MAY_*` 标志必须匹配操作：
- `MAY_READ`：读操作前
- `MAY_WRITE`：写操作前
- `MAY_EXEC`：目录遍历
- `MAY_SATTR`：setattr

常见错误：在 `vfs_write()` 前使用 `MAY_READ`（操作使用了错误的标志）。

**文件类型强制：** 当调用者假设特定文件类型时，将 `S_IFREG` / `S_IFDIR` 传递给 `fh_verify()`。使用 `0` 跳过检查。

请求范围的句柄在 args 结构体中由框架释放。COMPOUND 框架管理 cstate 句柄生命周期。

## NFSv4 Stateid 生命周期

Stateid 跟踪打开的文件、锁、委托和布局。每种类型有特定的加锁要求。

**Stateid 类型及其锁（用于 `sc_status`）：**
- Open/lock stateid：`cl_lock`（open stateid 还有 `st_mutex`）
- Delegation：`deleg_lock`
- Layout：`ls_lock`

**SC_STATUS_CLOSED：** 状态修改操作必须在适当锁下检查 `sc_status`，且检查与修改必须是原子的。

**代际编号：** 使用 `nfs4_inc_and_copy_stateid()` 而不是手动的 `si_generation` 递增。状态修改操作必须 bump generation。

**Delegation 回调：** 在 `nfsd4_run_cb()` 前需要 `refcount_inc(&dp->dl_stid.sc_count)` 以保持 delegation 存活。回调期间的客户端生命周期由 `cl_cb_inflight` 保护（在 `nfsd4_run_cb()` 内部递增），而不是 `cl_rpc_users`。`destroy_client()` 调用 `nfsd4_shutdown_callback()` 等待 `cl_cb_inflight` 耗尽。

**Stateid 文件验证：** Stateid 必须针对文件句柄进行验证：
- `CLAIM_DELEG_CUR` 必须通过 `fh_match()` 验证文件句柄是否匹配 `sc_file->fi_fhandle`
- Lock stateid 必须针对其 open stateid 的文件进行验证
- 仅通过 stateid 查找而不验证文件句柄会允许访问错误的文件

**第三方租约：** 在将 `fl_owner` 转换为 `nfs4_delegation` 前，检查 `fl->fl_lmops == &nfsd_lease_mng_ops`。非 NFSD 的租约有不同类型的 `fl_owner`。

**Write-attrs delegation：** `FMODE_NOCMTIME` 仅对 `OPEN_DELEGATE_WRITE_ATTRS_DELEG` 有效。在其他 delegation 类型上访问 `dl_atime` / `dl_mtime` 是无效的。

**多客户端 delegation 冲突：** 同客户端短路仍必须打破其他客户端的 delegation。`if (same_client) return` 而不打破其他客户端 delegation 是 bug。

**Delegation 释放期间的 VFS：** `nfs4_unlock_deleg_lease()` 中的 `notify_change()` 需要在 `ia.ia_valid` 中包含 `ATTR_DELEG`，以防止被释放的 delegation 重新被打破。

**Layout stateid：** `ls_lock` 是自旋锁。需要睡眠的操作（段操作、分配）应使用 `ls_mutex`。

最终 put 期间析构函数字段的访问是安全的（refcount 保证排他性）。

## 错误码映射

NFS 错误码必须在内部 errno 值和网络协议值之间正确映射。

- **NFSv3 状态：** 过程必须返回 `nfsd3_map_status(resp->status)`，而不是裸的 `rpc_success`。NFSv2 需要 `nfserrno()` 转换。
- **内部错误：** `PTR_ERR()` 值在到达 RPC 层前必须通过 `nfserrno()` 转换。负 errno 值不能到达网络。
- **版本特定错误：** 共享代码（vfs.c、nfsfh.c）中的 NFSv4 专用错误（如 `nfserr_delay`）从 v2/v3 路径到达时会引起问题。Session 错误不能从 pre-v4.1 路径返回。注意：NFSERR_INVAL 在 NFSv2（RFC 1094）中没有定义；`nfserr_file_open` 对非普通文件无效。
- **双重映射：** `nfserrno()` 将负 errno 转换为 NFS 状态。对已转换的 `__be32` 再次应用会破坏值。
- **EOPENSTALE：** 不要直接转换为 `nfserr_stale`。EOPENSTALE 表示需要在更高层重试。

## 锁层次

从外到内：
```
nn->client_lock → nn->deleg_lock → nn->s2s_cp_lock → fp->fi_lock → clp->cl_lock → stp->st_mutex
```

除 `st_mutex` 外都是自旋锁。`nn->nfsd_ssc_lock` 在主层次之外但与 `s2s_cp_lock` 互斥——永远不要同时持有两者。

**锁作用域：**
- 每命名空间（`nn->client_lock`）：`cl_time`、`cl_lru`、`cl_idhash`、`grace_ended`
- 每客户端（`clp->cl_lock`）：`cl_openowners`、`cl_sessions`、`cl_revoked`、`cl_flags`（包括 `NFSD4_CLIENT_RECLAIM_COMPLETE`）

**Stateid 查找中的 TOCTOU：** 在 `cl_lock` 下 `find_stateid_locked()` 和 `mutex_lock(&stp->st_mutex)` 之间的间隙需要后续的 `nfsd4_verify_open_stid()` 检查来检测并发取消哈希。

**VFS 回调路径：** `nfs4_put_stid()` 在 VFS break 回调下（持有 `flc_lock`）可能通过 `refcount_dec_and_lock()` 获取 `cl_lock`，导致死锁。当 refcount 不可能降到零时，使用 `refcount_dec()`。

## 客户端状态机

客户端状态（`cl_state` 枚举）：`NFSD4_ACTIVE` -> `NFSD4_COURTESY` -> `NFSD4_EXPIRABLE`。客户端确认通过 `cl_flags` 中的 `NFSD4_CLIENT_CONFIRMED` 位单独跟踪。

**有效转换：**
- `NFSD4_ACTIVE` -> `NFSD4_COURTESY`（租约过期，无状态冲突；`nfs4_get_client_reaplist`）
- `NFSD4_COURTESY` -> `NFSD4_ACTIVE`（客户端重连）
- `NFSD4_COURTESY` -> `NFSD4_EXPIRABLE`（冲突；`try_to_expire_client` 通过 `cmpxchg`）
- `NFSD4_EXPIRABLE` -> 销毁（laundromat 收回）

`NFSD4_EXPIRABLE` 不能回到 `NFSD4_ACTIVE`。只有 `NFSD4_COURTESY` 可以在重连时回到 `NFSD4_ACTIVE`。

**COURTESY 客户端：** 转换为 COURTESY 需要 laundromat 集成。`cl_time` 必须设置，laundromat 必须检查并在超时后过期。

**管理接口：** sysfs/procfs 写入必须持有 `nfsd_mutex` 或检查 `nn->nfsd_serv` 以避免 use-after-free 竞争。

**子 stateid：** 父销毁必须释放子 stateid（copynotify）。`release_openowner()` 必须调用 `nfs4_free_cpntf_statelist()`。

**双重初始化：** 从多个调用路径可达的初始化函数可能在第二次调用时触发 `BUG_ON`。

## 宽限期与租约管理

宽限期允许客户端在服务端重启后回收状态。

**宽限期内：**
- 非回收操作（OPEN、LOCK、改变大小的 SETATTR）如果会创建新状态，必须返回 `nfserr_grace`
- 回收使用 `CLAIM_PREVIOUS`、`CLAIM_DELEGATE_PREV`（打开操作），`lk_reclaim=true`（锁操作）
- 参考：RFC 8881 第 8.4 节

**租约计时：**
- 使用 `nn->nfsd4_lease` 和 `nn->nfsd4_grace` 作为持续时间，不要硬编码
- 使用 `ktime_get_boottime_seconds()` 作为 `cl_time`，不要用 `ktime_get()`——租约时间必须在挂起/恢复期间存活

**宽限期结束：**
- 必须考虑仍有回收进行中的客户端
- 从未发出 `RECLAIM_COMPLETE` 的客户端必须在宽限期结束后被销毁
- 客户端记录需要 `nfsd4_client_record_create()` 以跨崩溃持久化

宽限期内回收操作（`CLAIM_PREVIOUS`、`lk_reclaim`）是预期的。

## 用户命名空间 ID 转换

NFSD kthread 使用 `init_user_ns` 作为 `current_user_ns()`，这对容器化客户端是错误的。

**正确的命名空间来源：** 在请求路径中使用 `nfsd_user_namespace(rqstp)` 进行 `from_kuid()` / `from_kgid()`，而不是 `init_user_ns` 或 `current_user_ns()`。对于服务器间 socket 创建，使用 `nn->net` 而不是 `current->nsproxy->net_ns`。

**ID 验证：** `make_kuid()` / `make_kgid()` 结果需要 `uid_valid()` / `gid_valid()` 验证。将无效 ID 映射到 `GLOBAL_ROOT_UID` 是权限提升。

**ACL 编码：** 每个 ACL 条目需要 `from_kuid_munged(ns, ...)` 转换。未转换的原始 `kuid_t` 值会导致跨命名空间的权限问题。

**一致性：** 在同一操作中不要混用 `init_user_ns` 和 `nfsd_user_namespace()`。

仅主机的内部路径（模块初始化、procfs）可以使用 `init_user_ns`。idmap 路径在内部处理命名空间转换。

## 回调

回调是从服务端到客户端的异步 RPC。

**引用要求：** 回调期间的客户端生命周期由 `cl_cb_inflight` 管理，`nfsd4_run_cb()` 内部递增它——调用者不需要为回调分发递增 `cl_rpc_users`。Delegation 回调需要在 `nfsd4_run_cb()` 前递增 `sc_count` 以保持 delegation stid 存活。释放处理程序必须丢弃所有获取的引用。

**连接状态：** 在 `cl_lock` 下检查 `cl_cb_state == NFSD4_CB_UP`。只在相同锁持有下访问 `cl_cb_client`。

**序列号：** `se_cb_seq_nr`（在 `nfsd4_session` 上）的修改需要适当的加锁。无锁的并发递增会导致重复的序列号和 `BAD_SEQUENCE` 拒绝。

**客户端销毁：** `destroy_client()` 必须调用 `nfsd4_shutdown_callback()` 以耗尽正在进行的回调后再释放。

**NFSv4.0 兼容性：** `cl_cb_session` 对 NFSv4.0 客户端为 NULL。在访问 `cl_cb_session` 前检查 `cl_minorversion > 0`。

通过 RPC 完成处理程序的延迟释放是可接受的。

## 会话槽位（Session Slots）

Session（NFSv4.1+）使用 slot 进行请求排序和重放检测。

- **Slot 索引验证：** `xa_load(&session->se_slots, slotid)` 访问需要事先检查 `slotid < se_fchannel.maxreqs`。客户端提供的 slotid 没有边界检查会导致访问不存在的 slot。
- **Seqid 验证顺序：** 在修改前将请求 seqid 与 `sl_seqid` 比较。原始值区分：重放（匹配）、新请求（+1）、乱序（其他）。
- **重放安全：** 重放路径在返回缓存回复前必须检查 `same_creds()`。否则攻击者可以重放另一个客户端的响应。
- **Slot 排他性：** 在复合执行前设置 `NFSD4_SLOT_INUSE` 标志到 `sl_flags`，执行后清除。所有退出路径（错误、延迟）必须清除该标志。
- **Session 拆除：** 从哈希表移除 session 并排空活动复合操作后再释放 slot。
- **缓存回复生命周期：** 缓存的回复数据位于 `sl_data[]`（灵活数组）中，长度由 `sl_datalen` 指定。确保 session 拆除时正确失效缓存数据，以防止重放时的 use-after-free。

## 页数组管理

NFSD 管理用于读/写数据传输的页数组。

**关键指针：**
- `rq_pages`：页数组基址
- `rq_next_page`：下一个可用槽位
- `rq_page_end`：哨兵（末尾之后一个）
- `rq_maxpages`：数组大小

- **读过程：** 在读调用前从 `rqstp->rq_next_page` 保存 `resp->pages`。读操作会推进 `rq_next_page`，之后的引用会使用错误的指针。（修复 7978e9bea278）
- **边界检查：** 推进 `rq_next_page` 的循环需要 `rq_next_page < rq_page_end` 守卫。`rq_bvec` 索引需要 `rq_maxpages` 边界。（修复 e1b495d02c53、3be7f32878e7）
- **COMPOUND 页同步：** 单个 NFSv4 操作不得手动同步 `page_ptr` / `rq_next_page`。`nfsd4_encode_operation()` 在每个操作后集中同步 page_ptr——这是正确的模式。（修复 ed4a567a179e）
- **READDIR 回收：** READDIR 完成后，设置 `rqstp->rq_next_page = xdr.page_ptr + 1` 以回收未使用的页面。页计数：`(count + PAGE_SIZE - 1) >> PAGE_SHIFT`。（修复 3c86794ac0e6、76ed0dd96eeb）
- **Splice 延续：** 在 `nfsd_splice_actor()` 中，当 `page == *(rq_next_page - 1)` 且偏移量不是页对齐时，同一页正在被延续——不要再次添加。检查 `svc_rqst_replace_page()` 返回值。（修复 27c934dd8832、91e23b1c3982）

## 拷贝卸载（Copy Offload）

NFSv4.2 COPY 操作支持异步和服务端到服务端拷贝。

- **异步拷贝完成：** 在 `s2s_cp_lock` 下的 IDR 移除必须在最终 put 之前。陈旧的 IDR 条目允许在已释放的状态上执行 `OFFLOAD_STATUS` 查询。
- **取消：** `OFFLOAD_CANCEL` 需要在锁下进行原子状态转换。检查和取消之间的窗口允许与完成竞争，导致 use-after-free 或 double-free。
- **CB_OFFLOAD 顺序：** 在调用 `nfsd4_run_cb()` 前设置结果字段（`wr_bytes_written`、`wr_stable_how`）。回调可能立即读取结果。
- **S2S 凭据：** 服务器间拷贝缓存的 RPC 凭据必须在每个 chunk 前验证有效性。GSS 凭据可能在长时间拷贝期间过期。
- **COPY_NOTIFY 验证：** `nfsd4_setup_inter_ssc()` 必须验证 `cnr_stateid` 存在、属于请求客户端且未过期。
- **资源限制：** 异步拷贝提交需要每客户端或全局限制，以防止无界并发 COPY 操作导致内存耗尽。

同步拷贝直接使用复合作用域的引用（不需要额外引用）。

## 安全验证

- **验证绕过：** 在 `fh_verify()` 前的新分支或早期返回会跳过验证。访问 `fh_dentry` 的新辅助函数必须直接调用 `fh_verify()`，或文档化要求调用者事先验证的约定。
- **Stateid 验证：** NFSv4 有状态操作（read、write、lock、改变大小的 setattr）在文件访问前需要 `nfs4_preprocess_stateid_op()`。
- **跨导出操作：** RENAME 和 LINK 必须通过 `fh_verify()` 验证源和目标文件句柄。移除任一验证会导致未验证的访问。
- **伪文件系统暴露：** NFSv4 伪文件系统不得从 v2/v3 过程访问。`fh_verify()` 执行此版本门控。

调用者已经验证了 fh（有文档说明的约定）或 COMPOUND 框架预验证了 cstate 句柄的函数不需要重新验证。

## Netlink 接口

NFSD 使用 genetlink 进行配置（`include/uapi/linux/nfsd_netlink.h`、`fs/nfsd/netlink.h`）。

**策略要求：**
- 每个 `NFSD_A_*` 枚举需要对应的 `nla_policy` 条目
- 字符串属性需要 `NLA_NUL_STRING` 并带显式 `.len` 边界
- 在没有策略保证空终止的情况下，不要对字符串使用 `nla_data()`
- 嵌套属性需要自己的策略数组用于 `nla_parse_nested()`

由 genetlink 策略验证强制为必需的属性是安全的。

**特权检查：** 修改 NFSD 状态的处理程序在任何副作用之前需要 `capable(CAP_NET_ADMIN)` 或 `ns_capable()`。

**命名空间隔离：** 使用 `genl_info_net(info)` 获取网络命名空间，而不是 `&init_net` 或全局指针。使用 `ns_capable()` 进行命名空间相关的特权检查。

**状态同步：** 对 `nn->nfsd_serv` 或 NFSD 运行状态的检查必须受 `nfsd_mutex` 保护，直到后续的修改完成，以防止竞争。

## NFS 重新导出

当 NFSD 导出 NFS 挂载的文件系统时需要特殊处理。

- **文件句柄：** 在 NFS 超级块（`s_magic == NFS_SUPER_MAGIC`）上，`i_ino` 在上游重连后不稳定。`fh_compose()` 或文件句柄编码必须嵌入上游文件句柄（`NFS_FH()`）而不是使用 `i_ino`。
- **ESTALE 处理：** 不要对 NFS 支持的文件系统上的 `-ESTALE` 重试。ESTALE 表示句柄永久无效；重试无法解决。
- **锁顺序：** 在提交本地 NFSD 状态前先获取上游 VFS 锁（`vfs_lock_file()`）。失败的上游锁不得留下陈旧的本地状态。
- **挂载穿越：** `nfsd_cross_mnt()` 或 `follow_down()` 必须在穿越挂载前检查目标超级块是否为 NFS。重新导出需要不同的句柄编码和凭据处理。
- **双重宽限期：** 上游宽限期（来自 NFS 客户端的 `-EAGAIN`）和本地 NFSD 宽限期是独立的。两者都需要处理。
- **凭据双重映射：** 重新导出应用 squash/security 转换两次。重新导出上的 `no_root_squash` 配合上游的 `root_squash` 会引起问题。
- **导出 fsid：** 显式设置 `exp->ex_fsid` 或 `exp->ex_uuid`。从 NFS 挂载设备号派生的值在重新挂载时会变化。

本地文件系统导出（`s_magic != NFS_SUPER_MAGIC`）不需要重新导出处理。

## 资源限制

**每客户端限制在以下方面必需：**
- `nfs4_alloc_stid()`、`alloc_init_deleg()`
- `create_session()`
- 异步拷贝队列深度
- 工作队列项（每客户端 `cl_callback_wq`、`laundry_wq`）

**限制检查顺序：** 在分配前验证限制，而不是之后。分配->检查->在重负载下释放会导致瞬态 OOM。

**COMPOUND 边界：** 分发循环需要 `args->opcnt` 的上限。超出时返回 `nfserr_resource`。

**计数器泄漏：** 分配前 `atomic_inc()` 资源计数器（`num_delegations` 等）需要在所有错误路径上匹配 `atomic_dec()`。

**昂贵操作：** 限制 READDIR/GETATTR 的 `maxcount`。限制 ACL、安全标签、所有者名称 idmap 查找的每条目开销。

服务端生成的限制（如 `CREATE_SESSION` 后的 session slot 数）和受固定协议最大值限制的分配不需要额外检查。

## 代码风格

- 反向圣诞树变量排序（reverse-christmas tree）
- `nfs_ok` / `nfserr_*` 错误约定
- `cpu_to_be32` / `be32_to_cpu` 字节序转换
- 新的 NFSv4 XDR 代码应使用 `nfs4xdr_gen.c`（xdrgen）

## Invariants（不变式）

- `fh_dentry` 访问前必须有 `fh_verify()`，否则导致 NULL 解引用
- `nfsd4_run_cb()` 前必须持有所需引用，否则 use-after-free
- `xdr_reserve_space()` 返回值必须检查，否则 NULL 解引用
- Stateid 查找后必须 `fh_match()` 文件句柄，否则错误文件访问
- 请求路径中使用 `from_kuid()` 与 `init_user_ns` 会导致容器逃逸
- 违反锁层次会导致死锁
- `sc_status` 检查/修改必须在锁下原子进行
- 状态变更必须在编码成功确认后，否则重试时损坏
- Session slot 索引必须检查 `maxreqs`，否则无效 slot 访问
- 异步拷贝 IDR 移除必须在最终 put 之后使用，否则过期状态查询

## 需要专家评审的触发条件

变更触及以下内容时标记为需专家评审：XDR 原语或基础设施；引用计数原语；`fh_verify()` 语义；stateid 生命周期；锁顺序；客户端状态机或宽限期逻辑；回调分发/完成/重试；session slot 或 SEQUENCE 处理；新的 RPC 过程或 NFSv4 操作；命名空间转换路径；页数组或 splice 基础设施；拷贝卸载生命周期或 S2S 认证；genetlink 族或策略定义；资源限制或分配模式；重新导出或跨挂载处理；变更超过 100 行触及多个核心文件。

## Quick Checks（快速检查清单）

- [ ] `fh_dentry` 访问前是否调用了 `fh_verify()`？
- [ ] `nfsd4_run_cb()` 前是否持有所需引用？
- [ ] `xdr_reserve_space()` 返回值是否检查了 NULL？
- [ ] Stateid 查找后是否使用了 `fh_match()` 验证文件句柄？
- [ ] 请求路径中是否使用了正确的命名空间（`nfsd_user_namespace()`）？
- [ ] 锁获取是否遵循层次结构？
- [ ] `sc_status` 的检查和修改是否在锁下原子进行？
- [ ] 状态变更是否在编码成功确认后才进行？
- [ ] Session slot 索引是否对 `maxreqs` 进行了边界检查？
- [ ] 异步拷贝的 IDR 移除是否在最终 put 之前？
