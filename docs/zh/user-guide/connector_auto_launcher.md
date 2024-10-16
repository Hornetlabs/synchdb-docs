# 连接器自动启动器

## 启用 SynchDB 自动启动器
当对特定的`连接器名称`执行 `synchdb_start_engine_bgw()` 时，连接工作进程将有资格自动启动。同样，当执行 `synchdb_stop_engine_bgw()` 时，它将失去自动启动资格。

可以通过以下方式启用自动连接器启动器：

* 在 postgresql.conf 中将 `synchdb` 添加到 `shared_preload_libraries` GUC 选项
* 在 postgresql.conf 中将新的 GUC 选项 `synchdb.synchdb_auto_launcher` 设置为 true
* 重启 PostgreSQL 服务器以使更改生效

例如：
```
shared_preload_libraries = 'synchdb'
synchdb.synchdb_auto_launcher = true
```
在启动时，SynchDB 扩展将被很早预加载。当 `synchdb.synchdb_auto_launcher` 设置为 true 时，SynchDB 将生成一个 `synchdb_auto_launcher` 后台工作进程，该进程将检索 `synchdb_conninfo` 表中所有标记为 `active`（`isactive` 标志设置为 `true`）的连接信息。然后，它将以与调用 `synchdb_start_engine_bgw()` 相同的方式自动启动它们，每个连接器作为一个单独的后台工作进程。之后，`synchdb_auto_launcher` 将退出。

## 已知问题
`synchdb_auto_launcher` 工作进程将登录到默认的 `postgres` 数据库，并尝试从 `synchdb_conninfo` 表中查找活跃的连接器。如果 SynchDB 是从非默认数据库安装的，那么 `synchdb_auto_launcher` 将无法找到该表，因此无法自动启动连接器工作进程。未来，我们会使用 `synchdb_auto_launcher` 检查所有数据库，并根据每个数据库的 `synchdb_conninfo` 表自动启动连接器工作进程。
