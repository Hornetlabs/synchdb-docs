# Configuration

SynchDB supports the following GUC variables in postgresql.conf:

| GUC Variable| Type | Default Value | Description |
|-|-|-|-|
| synchdb.naptime | integer | 500 | The delay in milliseconds between each data polling from Debezium runner engine |
| synchdb.dml_use_spi | boolean | false | Option to use SPI to handle DML operations |
| synchdb.synchdb_auto_launcher | boolean | true | Option to automatically launch active SynchDB connector workers. This option only works when SynchDB is included in `shared_preload_library` GUC option |

## Technical Notes

- GUC (Grand Unified Configuration) variables are global configuration parameters in PostgreSQL
- Values are set in the `postgresql.conf` file
- Changes require a server restart to take effect
- `shared_preload_library` is a critical system configuration that determines which libraries are loaded at startup

## Configuration Examples

```conf
# Example configuration in postgresql.conf
synchdb.naptime = 1000               # Increase wait time to 1 second
synchdb.dml_use_spi = true           # Enable SPI usage for DML operations
synchdb.synchdb_auto_launcher = true # Enable automatic connector startup
```

## Usage Recommendations

1. **synchdb.naptime**
    - Lower values: Higher update frequency but more system load
    - Higher values: Lower system load but less frequent updates
    - Adjust based on data latency requirements

2. **synchdb.dml_use_spi**
    - Enable if specific SPI integration is needed
    - Keep `false` for standard DML operations

3. **synchdb.synchdb_auto_launcher**
    - Recommended to keep `true` for automatic management
    - Change to `false` only if manual connector control is required

## Performance Considerations

- Adjust `naptime` based on system load and latency requirements
- Monitor system performance when modifying these configurations
- Consider system resource impact when enabling additional features

## Common Use Cases

### High-Throughput Systems
```conf
synchdb.naptime = 100           # Faster polling for real-time updates
synchdb.dml_use_spi = false     # Standard DML for better performance
```

### Resource-Constrained Systems
```conf
synchdb.naptime = 1000          # Reduced polling frequency
synchdb.dml_use_spi = false     # Minimize additional overhead
```

### Development/Testing
```conf
synchdb.naptime = 500           # Default polling
synchdb.dml_use_spi = true      # Enable advanced features for testing
```

## Troubleshooting

1. **High CPU Usage**
    - Increase `synchdb.naptime`
    - Review DML operation patterns

2. **Data Latency Issues**
    - Decrease `synchdb.naptime`
    - Check network connectivity

3. **Startup Problems**
    - Verify `shared_preload_library` configuration
    - Check connector worker status

## Best Practices

1. **Initial Setup**
    - Start with default values
    - Monitor system performance
    - Adjust gradually based on requirements

2. **Production Environment**
    - Document all configuration changes
    - Test changes in staging first
    - Maintain backup of working configurations

3. **Monitoring**
    - Track system resource usage
    - Monitor data synchronization latency
    - Log configuration changes