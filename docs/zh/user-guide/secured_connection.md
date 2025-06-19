# 安全连接

## **配置安全连接**

为了确保与远程数据库的连接安全，我们需要为 `synchdb_add_conninfo` 创建的连接器配置额外的 SSL 相关参数。SSL 证书和私钥必须打包为 Java 密钥库文件，并附带密码。这些信息随后会通过 synchdb_add_extra_conninfo() 传递给 SynchDB。

## **synchdb_add_extra_conninfo**

**用途**：为 `synchdb_add_conninfo` 创建的现有连接器配置额外的连接器参数

| 参数 | 描述 | 必需 | 示例 | 备注 |
|:-:|:-|:-:|:-|:-|
| `name` | 此连接器的唯一标识符 | ✓ | `'mysqlconn'` | 必须在所有连接器中唯一 |
| `ssl_mode` | SSL 模式 | ☐ | `'verify_ca'` |可以是以下之一：<br><ul><li>“disabled”- 不使用 SSL。</li><li>“preferred”- 如果服务器支持，则使用 SSL。</li><li>“required”- 必须使用 SSL 建立连接。</li><li>“verify_ca”- 连接器与服务器建立 TLS，并将根据配置的信任库验证服务器的 TLS 证书。</li><li>“verify_identity”- 与 verify_ca 行为相同，但它还会检查服务器证书的通用名称以匹配系统的主机名。|
| `ssl_keystore` | 密钥库路径 | ☐ | `/path/to/keystore` | 密钥库文件的路径 |
| `ssl_keystore_pass` | 密钥库密码 | ☐ | `'mykeystorepass'` | 访问密钥库文件的密码 |
| `ssl_truststore` | 信任库路径 | ☐ | `'/path/to/truststore'` | 信任库文件路径 |
| `ssl_truststore_pass` | 信任库密码 | ☐ | `'mytruststorepass'` | 访问信任库文件的密码 |

```sql
SELECT synchdb_add_extra_conninfo('mysqlconn', 'verify_ca', '/path/to/keystore', 'mykeystorepass', '/path/to/truststore', 'mytruststorepass');
```

## **synchdb_del_extra_conninfo**

**用途**：删除由 `synchdb_add_extra_conninfo` 创建的额外连接器参数
```sql
SELECT synchdb_del_extra_conninfo('mysqlconn');
```