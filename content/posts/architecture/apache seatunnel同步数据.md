---
title: "apache seatunnel同步数据"
date: "2025-07-14"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

使用SeaTunnel提供的Java API来动态构建配置文件并执行 Job，获取到增量数据后，计算HMAC并写入到目标数据库。  
后续判断数据有没有被篡改时，再次计算HMAC并与目标数据库中的值进行比较。  
job是用java api的方式动态注册到seatunnel的job manager中  
seatunnel有http接口去开启和关闭job  
获取数据后发到kafka中，后续从kafka中读取数据进行HMAC计算和验证。  

```java
@Service
public class DynamicJobService {

    public String createDynamicJob(DynamicJobRequest request) throws Exception {
        // 动态构建配置
        String configContent = buildDynamicConfig(request);

        // 执行 Job
        return executeSeaTunnelJob(configContent, request.getJobId());
    }

    private String buildDynamicConfig(DynamicJobRequest request) {
        return String.format("""
                        env {
                          execution.parallelism = %d
                          job.mode = "%s"
                          execution.checkpoint.interval = %d
                        }
                        
                        source {
                          MySQL-CDC {
                            hostname = "%s"
                            port = %d
                            username = "%s"
                            password = "%s"
                            database-names = ["%s"]
                            table-names = [%s]
                            startup.mode = "%s"
                            server-id = %s
                          }
                        }
                        
                        sink {
                          Kafka {
                            bootstrap.servers = "%s"
                            topic = "%s"
                            format = "json"
                          }
                        }
                        """,
                request.getParallelism(),
                request.getJobMode(),
                request.getCheckpointInterval(),
                request.getMysqlHost(),
                request.getMysqlPort(),
                request.getMysqlUser(),
                request.getMysqlPassword(),
                request.getDatabaseName(),
                formatTableNames(request.getTables()),
                request.getStartupMode(),
                request.getServerId(),
                request.getKafkaServers(),
                request.getKafkaTopic()
        );
    }

    private String formatTableNames(List<String> tables) {
        return tables.stream()
                .map(table -> "\"" + table + "\"")
                .collect(Collectors.joining(", "));
    }

    private String executeSeaTunnelJob(String configContent, String jobId) throws Exception {
        ClientCommandArgs commandArgs = new ClientCommandArgs();
        commandArgs.setConfigFile(configContent);
        commandArgs.setEngineType(EngineType.FLINK);

        // 异步执行
        CompletableFuture.runAsync(() -> {
            try {
                SeaTunnel.run(commandArgs);
            } catch (Exception e) {
                logger.error("Job {} failed", jobId, e);
            }
        });

        return jobId;
    }
}
```



### 自定义 JDBC 增量源 (`IncrementalJdbcSource.java`)

```java
import org.apache.seatunnel.connector.jdbc.source.JdbcSource;
import org.apache.seatunnel.api.configuration.Option;
import org.apache.seatunnel.api.configuration.Options;
import org.apache.seatunnel.api.configuration.ReadonlyConfig;
import org.apache.seatunnel.api.source.Boundedness;
import org.apache.seatunnel.api.source.SeaTunnelSource;
import org.apache.seatunnel.api.source.SourceReader;
import org.apache.seatunnel.api.source.SourceSplit;
import org.apache.seatunnel.api.source.SupportColumnProjection;
import org.apache.seatunnel.api.source.SupportParallelism;
import org.apache.seatunnel.api.table.catalog.CatalogTable;
import org.apache.seatunnel.api.table.catalog.TablePath;
import org.apache.seatunnel.api.table.type.SeaTunnelDataType;
import org.apache.seatunnel.api.table.type.SeaTunnelRow;
import org.apache.seatunnel.api.table.type.SeaTunnelRowType;
import org.apache.seatunnel.common.constants.PluginType;
import org.apache.seatunnel.connectors.seatunnel.common.multitablesink.MultiTableResourceManager;

import java.util.List;

/**
 * 自定义增量 JDBC 源，继承自 JdbcSource 并添加增量处理能力
 */
public class IncrementalJdbcSource implements SeaTunnelSource<SeaTunnelRow, SourceSplit, ?>,
        SupportParallelism, SupportColumnProjection {
    
    // 定义增量同步相关的配置选项
    public static final Option<String> INCREMENTAL_COLUMN = Options.key("incremental_column")
            .stringType()
            .noDefaultValue()
            .withDescription("用于增量同步的时间戳或序列号字段名");

    public static final Option<Long> INITIAL_OFFSET = Options.key("initial_offset")
            .longType()
            .defaultValue(0L)
            .withDescription("初始偏移量（时间戳或序列号）");

    private ReadonlyConfig config;
    private JdbcSource jdbcSourceDelegate;
    
    @Override
    public String getPluginName() {
        return "IncrementalJdbc";
    }

    @Override
    public void prepare(ReadonlyConfig pluginConfig) {
        this.config = pluginConfig;
        // 使用标准的 JdbcSource 作为委托实现
        this.jdbcSourceDelegate = new JdbcSource();
        this.jdbcSourceDelegate.prepare(pluginConfig);
    }

    @Override
    public Boundedness getBoundedness() {
        return Boundedness.BOUNDED; // 或 UNBOUNDED，取决于具体需求
    }

    @Override
    public SeaTunnelDataType<SeaTunnelRow> getProducedType() {
        return jdbcSourceDelegate.getProducedType();
    }

    @Override
    public SourceReader<SeaTunnelRow, SourceSplit> createReader(SourceReader.Context readerContext) throws Exception {
        return jdbcSourceDelegate.createReader(readerContext);
    }

    @Override
    public List<CatalogTable> getProducedCatalogTables() {
        return jdbcSourceDelegate.getProducedCatalogTables();
    }
}
```

对于大数据量场景，考虑分片并行处理策略。

