# Configuration
SynchDB supports the following GUC variables in postgresql.conf:

| GUC                           	| type    	| default value 	| description                                                                                                                                             	|
|-------------------------------	|---------	|---------------	|---------------------------------------------------------------------------------------------------------------------------------------------------------	|
| synchdb.naptime               	| integer 	| 500           	| The delay in milliseconds between each data polling from Debezium runner engine                                                                         	|
| synchdb.dml_use_spi           	| boolean 	| false         	| Option to use SPI to handle DML operations                                                                                                              	|
| synchdb.synchdb_auto_launcher 	| boolean 	| true          	| Option to automatically launch active SynchDB connector workers. This option only works when SynchDB is included in `shared_preload_library` GUC option 	|
