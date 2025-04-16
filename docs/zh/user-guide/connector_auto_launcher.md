---
weight: 70
---
# 自动启动器

## **启用 SynchDB 自动启动器**
当对特定 `connector name` 执行 `synchdb_start_engine_bgw()` 时，连接工作进程将具备自动启动资格。同样，当执行 `synchdb_stop_engine_bgw()` 时，将取消此资格。

启用自动连接器启动器需要：

* 在 postgresql.conf 中将 `synchdb` 添加到 `shared_preload_libraries` GUC 选项
* 在 postgresql.conf 中将新的 GUC 选项 `synchdb.synchdb_auto_launcher` 设置为 true
* 重启 PostgreSQL 服务器使更改生效

示例：
```
shared_preload_libraries = 'synchdb'
synchdb.synchdb_auto_launcher = true
```

在启动时，SynchDB 扩展将很早被预加载。当 `synchdb.synchdb_auto_launcher` 设置为 true 时，SynchDB 将产生一个 `synchdb_auto_launcher` 后台工作进程，该进程将检索 `synchdb_conninfo` 表中标记为 `active` 的所有连接信息（`isactive` 标志设置为 `true`）。然后，它将以与调用 `synchdb_start_engine_bgw()` 相同的方式自动启动它们作为单独的后台工作进程。之后 `synchdb_auto_launcher` 将退出。

## **已知问题**
`synchdb_auto_launcher` 工作进程将登录到默认的 `postgres` 数据库，并尝试从 `synchdb_conninfo` 表中查找活动连接器。如果 SynchDB 安装在非默认数据库中，则 `synchdb_auto_launcher` 将无法找到该表，因此无法自动启动连接器工作进程。将来，我们将使 `synchdb_auto_launcher` 检查所有数据库，并根据每个数据库的 `synchdb_conninfo` 表自动启动连接器工作进程。

有关详细信息和更新，请查看[[Issue #71]](https://github.com/Hornetlabs/synchdb/issues/71) 。