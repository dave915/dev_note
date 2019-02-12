### DB 단순 조회시 Slave DB 보도록 수정
통계나 로그등 단순 조회시 수행할 메소드에 @Transactional(readOnly = true) 사용시 transaction이 readonly로 되면서 slave db를 보게됨.

Transaction을 시작할 때 트랜잭션 정보를 TransactionSynchronizationManager와 동기화 하기 전에 먼저 데이터소스에서 커넥션을 맺는데, LazyConnectionDataSourceProxy를 사용하면 동기화를 마친 뒤에 커넥션을 맺도록 실제 커넥션 맺는 시점을 늦출 수 있음.

(참고 : http://kwonnam.pe.kr/wiki/java/jdbc/replication)

DatabaseConfig.java
```java
@Bean
@ConfigurationProperties(prefix = "spring.datasource")
public DataSource dataSource() {
    return DataSourceBuilder.create().build();
}

@Bean
@ConfigurationProperties(prefix = "spring.readonly.datasource")
public DataSource readonlyDataSource() {
    return DataSourceBuilder.create().build();
}

@Bean
public DataSource routingDataSource(@Qualifier("dataSource") DataSource dataSource, @Qualifier("readonlyDataSource") DataSource readonlyDataSource) {
    ReplicationRoutingDataSource routingDataSource = new ReplicationRoutingDataSource();

    Map<Object, Object> dataSourceMap = new HashMap<Object, Object>();
    dataSourceMap.put("write", dataSource);
    dataSourceMap.put("read", readonlyDataSource);
    routingDataSource.setTargetDataSources(dataSourceMap);
    routingDataSource.setDefaultTargetDataSource(dataSource);

    return routingDataSource;
}

@Primary
@Bean(name = "routedDataSource")
public DataSource routedDataSource() {
    return new LazyConnectionDataSourceProxy(routingDataSource(dataSource(), readonlyDataSource()));
}
```
ReplicationRoutingDataSource.java
```java
@Slf4j
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        String dataSourceType = TransactionSynchronizationManager.isCurrentTransactionReadOnly() ? "read" : "write";
        log.debug("current dataSourceType : {}", dataSourceType);
        return dataSourceType;
    }
}
```
