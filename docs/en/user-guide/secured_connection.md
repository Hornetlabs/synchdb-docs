# Secured Connection

## **Configure Secured Connection**

to secure the connection to remote database, we need to configure additional SSL related parameters to a connector that has been created by `synchdb_add_conninfo`. The SSL certificates and private keys must be packaged as Java keystore file with a passphrase. These information is then passed to SynchDB via synchdb_add_extra_conninfo().


## **synchdb_add_extra_conninfo**

**Purpose**: Configures extra connector parameters to an existing connector created by `synchdb_add_conninfo`

| Parameter | Description | Required | Example | Notes |
|:-:|:-|:-:|:-|:-|
| `name` | Unique identifier for this connector | ✓ | `'mysqlconn'` | Must be unique across all connectors |
| `ssl_mode` | SSL mode | ☐ | `'verify_ca'` | can be one of: <br><ul><li> "disabled" - no SSL is used. </li><li> "preferred" - SSL is used if server supports it. </li><li> "required" - SSL must be used to establish a connection. </li><li> "verify_ca" - connector establishes TLS with the server and will also verify server's TLS certificate against configured truststore. </li><li> "verify_identity" - same behavior as verify_ca but it also checks the server certificate's common name to match the hostname of the system. |
| `ssl_keystore` | keystore path | ☐ | `/path/to/keystore` | path to the keystore file |
| `ssl_keystore_pass` | keystore password | ☐ | `'mykeystorepass'` | password to access the keystore file |
| `ssl_truststore` | trust store path | ☐ | `'/path/to/truststore'` | path to the truststore file |
| `ssl_truststore_pass` | trust store password | ☐ | `'mytruststorepass'` | password to access the truststore file |


```sql
SELECT synchdb_add_extra_conninfo('mysqlconn', 'verify_ca', '/path/to/keystore', 'mykeystorepass', '/path/to/truststore', 'mytruststorepass');
```

## **synchdb_del_extra_conninfo**

**Purpose**: Deletes extra connector paramters created by `synchdb_add_extra_conninfo`
```sql
SELECT synchdb_del_extra_conninfo('mysqlconn');
```