# Lanzador Automático de Conectores

## Habilitar el Lanzador Automático de SynchDB
Un trabajador de conexión se vuelve elegible para el lanzamiento automático cuando se emite `synchdb_start_engine_bgw()` en un `connector name` específico. Del mismo modo, se vuelve inelegible cuando se emite `synchdb_stop_engine_bgw()`.

El lanzador automático de conectores se puede habilitar mediante:

* Agregar `synchdb` a la opción GUC `shared_preload_libraries` en postgresql.conf
* Establecer la nueva opción GUC `synchdb.synchdb_auto_launcher` en true en postgresql.conf
* Reiniciar el servidor PostgreSQL para que los cambios surtan efecto

Por ejemplo:
```conf
shared_preload_libraries = 'synchdb'
synchdb.synchdb_auto_launcher = true
```

Al inicio, la extensión SynchDB se precargará muy temprano. Con `synchdb.synchdb_auto_launcher` establecido en true, SynchDB generará un trabajador en segundo plano `synchdb_auto_launcher` que recuperará todas las conexiones en la tabla `synchdb_conninfo` marcadas como `active` (tiene la bandera `isactive` establecida en `true`). Luego, los iniciará automáticamente como un trabajador en segundo plano separado de la misma manera que cuando se llama a `synchdb_start_engine_bgw()`. `synchdb_auto_launcher` saldrá después.

## Problema Conocido
El trabajador `synchdb_auto_launcher` iniciará sesión en la base de datos `postgres` predeterminada e intentará encontrar conectores activos en la tabla `synchdb_conninfo`. Si SynchDB se ha instalado desde una base de datos no predeterminada, entonces `synchdb_auto_launcher` no podrá encontrar la tabla y, por lo tanto, no iniciará automáticamente el trabajador del conector. En el futuro, haremos que `synchdb_auto_launcher` verifique todas las bases de datos e inicie automáticamente los trabajadores del conector basándose en las tablas `synchdb_conninfo` de cada base de datos.

Para más información y actualizaciones, consulte [[Issue #71]](https://github.com/Hornetlabs/synchdb/issues/71).
