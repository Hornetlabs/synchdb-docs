---
weight: 40
---
# 配置说明

SynchDB 在 postgresql.conf 中支持以下 GUC 变量：

| GUC 变量 | 类型 | 默认值 | 描述 |
|-|-|-|-|
| synchdb.naptime | integer | 500 | 从 Debezium runner 引擎轮询数据的时间间隔（毫秒） |
| synchdb.dml_use_spi | boolean | false | 是否使用 SPI 处理 DML 操作 |
| synchdb.synchdb_auto_launcher | boolean | true | 是否自动启动活跃的 SynchDB 连接器工作进程。此选项仅在 SynchDB 被包含在 `shared_preload_library` GUC 选项中时生效 |

## 技术说明

- GUC（Grand Unified Configuration）是 PostgreSQL 中的全局配置参数
- 这些值需要在 `postgresql.conf` 文件中设置
- 修改配置后需要重启服务器才能生效
- `shared_preload_library` 是一个关键的系统配置，决定了启动时加载哪些库

## 配置示例

```conf
# postgresql.conf 配置示例
synchdb.naptime = 1000               # 将等待时间增加到1秒
synchdb.dml_use_spi = true           # 启用 SPI 处理 DML 操作
synchdb.synchdb_auto_launcher = true # 启用自动连接器启动
```

## 使用建议

1. **synchdb.naptime**
    - 较低的值：更新频率更高，但系统负载更大
    - 较高的值：系统负载较低，但更新频率降低
    - 根据数据延迟要求进行调整

2. **synchdb.dml_use_spi**
    - 需要特定 SPI 集成时启用
    - 标准 DML 操作保持 `false`

3. **synchdb.synchdb_auto_launcher**
    - 建议保持 `true` 以实现自动管理
    - 仅在需要手动控制连接器时改为 `false`

## 性能考虑

- 根据系统负载和延迟要求调整 `naptime`
- 修改配置时监控系统性能
- 启用额外功能时考虑系统资源影响

## 常见使用场景

### 高吞吐量系统
```conf
synchdb.naptime = 100           # 更快的轮询以实现实时更新
synchdb.dml_use_spi = false     # 标准 DML 以获得更好的性能
```

### 资源受限系统
```conf
synchdb.naptime = 1000          # 降低轮询频率
synchdb.dml_use_spi = false     # 最小化额外开销
```

### 开发/测试环境
```conf
synchdb.naptime = 500           # 默认轮询间隔
synchdb.dml_use_spi = true      # 启用高级功能进行测试
```

## 故障排除

1. **CPU 使用率高**
    - 增加 `synchdb.naptime` 值
    - 检查 DML 操作模式

2. **数据延迟问题**
    - 减少 `synchdb.naptime` 值
    - 检查网络连接状况

3. **启动问题**
    - 验证 `shared_preload_library` 配置
    - 检查连接器工作进程状态

## 最佳实践

1. **初始设置**
    - 从默认值开始
    - 监控系统性能
    - 根据需求逐步调整

2. **生产环境**
    - 记录所有配置更改
    - 在测试环境中先测试更改
    - 保持正常工作配置的备份

3. **监控**
    - 跟踪系统资源使用情况
    - 监控数据同步延迟
    - 记录配置更改

## 补充说明

1. **性能优化**
    - 在高负载情况下适当增加 `naptime`
    - 需要实时性时可以降低 `naptime`，但要注意系统负载
    - 建议根据实际业务场景进行压力测试

2. **安全考虑**
    - 定期检查配置文件权限
    - 记录配置更改日志
    - 保持配置文件的备份

3. **维护建议**
    - 定期检查系统日志
    - 监控连接器状态
    - 制定配置变更流程