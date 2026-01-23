#  Monitoring & Production Considerations

## Kafka Monitoring Configuration

```
# application-prod.yml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BROKERS:localhost:9092}
    producer:
      acks: all
      retries: 3
      enable-idempotence: true
    consumer:
      group-id: le-alert-${HOSTNAME}
      auto-offset-reset: latest
      enable-auto-commit: false
      max-poll-records: 100

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,kafka
  metrics:
    export:
      prometheus:
        enabled: true
```
## Health Checks

```
// KafkaHealthIndicator.java
@Component
public class KafkaHealthIndicator implements HealthIndicator {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Override
    public Health health() {
        try {
            // Test Kafka connectivity
            kafkaTemplate.send("health-check", "ping").get(5, TimeUnit.SECONDS);
            return Health.up()
                .withDetail("brokers", kafkaTemplate.getProducerFactory()
                    .getBootstrapServers())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .withDetail("error", "Kafka unavailable")
                .build();
        }
    }
}
```

##  Production Best Practices


