### DROP TABLE
Propiedades capturadas:
- Nombre de tabla (en formato FQN) a eliminar

### ALTER TABLE ADD COLUMN
Captura las siguientes propiedades:

| Propiedad | Descripción |
|----------|-------------|
| Column Names | Nombres de las columnas recién añadidas |
| Data Types | Tipos de datos para nuevas columnas |
| Data Length | Especificaciones de longitud (si aplica) |
| Unsigned Flag | Restricciones unsigned |
| Nullability | Especificaciones NULL/NOT NULL |
| Default Values | Expresiones de valor predeterminado |
| Primary Keys | Definiciones actualizadas de clave primaria |

Otras propiedades que se pueden especificar durante ALTER TABLE ADD COLUMN no están soportadas en este momento.

### ALTER TABLE DROP COLUMN
Captura:
- Lista de nombres de columnas a eliminar

### ALTER TABLE ALTER COLUMN
Modificaciones soportadas:

| Modificación | Descripción |
|--------------|-------------|
| Data Type | Cambiar tipo de dato de columna |
| Type Length | Modificar longitud/precisión del tipo |
| Default Value | Alterar/eliminar valores predeterminados |
| NOT NULL | Modificar/eliminar restricción NOT NULL |

Otras propiedades que se pueden especificar durante ALTER TABLE ALTER COLUMN no están soportadas en este momento.

Tenga en cuenta que SynchDB solo soporta cambios básicos de tipo de dato en una columna existente. Por ejemplo, `INT` → `BIGINT` o `VARCHAR` → `TEXT`. Los cambios complejos de tipo de dato como `TEXT` → `INT` o `INT` → `TIMESTAMP` no están soportados actualmente. Esto se debe a que PostgreSQL requiere que el usuario proporcione adicionalmente una función de conversión de tipo para realizar la conversión como resultado del cambio complejo de tipo de dato. SynchDB actualmente no tiene conocimiento de qué funciones de conversión de tipo usar para conversiones específicas. En el futuro, podríamos permitir que el usuario proporcione sus propias funciones de conversión para usar en conversiones específicas a través del archivo de reglas, pero por ahora, no está soportado.

## Comportamiento Específico de Base de Datos
### Eventos de Cambio DDL en MySQL
Dado que MySQL registra tanto las operaciones DDL como DML en el binlog, SynchDB puede replicar tanto DDLs como DMLs cuando ocurren. No se necesitan acciones especiales en el lado de MySQL para habilitar la replicación de DDLs.

### Eventos de Cambio DDL en SQLServer 
SQLServer no soporta de forma nativa la replicación DDL en modo streaming. El esquema de tabla es construido por SynchDB durante la fase de construcción de instantánea inicial cuando el conector se inicia por primera vez. Después de esta fase, SynchDB intentará detectar cualquier cambio de esquema, pero necesitan ser agregados explícitamente a la lista de tablas CDC de SQL Server.

#### Disparar evento CREATE TABLE en SQLServer
Para crear una nueva tabla en SQL Server y agregarla a su lista de tablas CDC:
```
CREATE TABLE dbo.altertest (a int, b text);
GO

EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance='dbo_altertest_1
GO
```

El comando agrega la tabla `dbo.altertest` a la lista de tablas CDC y causaría que SynchDB reciba un evento de cambio DDL CREATE TABLE.

#### Disparar Eventos ALTER TABLE
Si se altera una tabla existente (agregar, eliminar o alterar columna), necesita ser actualizada explícitamente en la lista de tablas CDC de SQLServer, para que SynchDB pueda recibir los eventos ALTER TABLE.

Por ejemplo:

Alterar una tabla en SQLServer:
```
ALTER TABLE altertest ADD c NVARCHAR(MAX), d INT DEFAULT 0 NOT NULL, e NVARCHAR(255) NOT NULL, f INT DEFAULT 5 NOT NULL, CONSTRAINT PK_altertest PRIMARY KEY (d, f);
GO
```

Deshabilitar la instancia de captura anterior:
```
EXEC sys.sp_cdc_disable_table @source_schema='dbo', @source_name='altertest', @capture_instance='dbo_altertest_1';
GO
```

Habilitar como nueva instancia de captura:
```
EXEC sys.sp_cdc_enable_table @source_schema = 'dbo', @source_name = 'altertest', @role_name = NULL, @supports_net_changes = 0, @capture_instance = 'dbo_altertest_2';
GO
```

Agregar nuevo registro:
```
INSERT INTO altertest VALUES(1, 's', 'c', 1, 'v', 5);
GO
```

El ejemplo anterior debería permitir que SynchDB reciba eventos ALTER TABLE ADD COLUMN e INSERT. No hay necesidad de reiniciar SQL Server o SynchDB para capturar tales eventos. Lo mismo aplica para los eventos ALTER TABLE DROP COLUMN y ALTER TABLE ALTER COLUMN también.