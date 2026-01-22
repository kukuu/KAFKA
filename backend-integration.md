# Backend Integration

## Spring Kafka Configuration**
@Configuration: 

@EnableKafka: 


```
// KafkaConfig.java
@Configuration 
@EnableKafka 

public class KafkaConfig {

    @Bean
    public ProducerFactory<String, Alert> alertProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, Alert> alertKafkaTemplate() {
        return new KafkaTemplate<>(alertProducerFactory());
    }

    @Bean
    public ConsumerFactory<String, Alert> alertConsumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "alert-correlation-group");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "com.le.correlation.model");
        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Alert> alertListenerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Alert> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(alertConsumerFactory());
        return factory;
    }
}
```

- What this does:
1. @Configuration //Marks this as a Spring configuration class
2. @EnableKafka //Enables Spring's Kafka support framework
//Purpose: Both above creates a centralized configuration for Kafka messaging in the application.

- Producer Factory
1. Creates a factory that produces Kafka message senders
2. Key-Type: String (message key)
3. Value-Type: Alert (message payload - your custom object)

- Configuration Settings:

Setting	| Value	| Purpose

BOOTSTRAP_SERVERS_CONFIG | 	localhost:9092 | 	Kafka broker location

KEY_SERIALIZER_CLASS_CONFIG | 	StringSerializer.class | 	Converts String keys to bytes

VALUE_SERIALIZER_CLASS_CONFIG | 	JsonSerializer.class |	Converts Alert objects to JSON

_Important: JsonSerializer automatically serializes Java objects to JSON format._

## Topic Configuration

```
// KafkaTopicConfig.java
@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic rawAlertsTopic() {
        return TopicBuilder.name("alerts.raw")
                .partitions(3)
                .replicas(1)
                .config(TopicConfig.RETENTION_MS_CONFIG, "604800000") // 7 days
                .build();
    }

    @Bean
    public NewTopic processedAlertsTopic() {
        return TopicBuilder.name("alerts.processed")
                .partitions(3)
                .replicas(1)
                .build();
    }
}
```
## Alert Producer Service
```
// AlertProducer.java
@Service
@Slf4j
public class AlertProducer {

    private static final String RAW_ALERTS_TOPIC = "alerts.raw";
    private static final String PROCESSED_ALERTS_TOPIC = "alerts.processed";

    @Autowired
    private KafkaTemplate<String, Alert> kafkaTemplate;

    public void sendRawAlert(Alert alert) {
        log.info("Producing raw alert: {}", alert.getId());
        
        kafkaTemplate.send(RAW_ALERTS_TOPIC, alert.getId(), alert)
            .addCallback(
                result -> log.info("Alert sent successfully: {}", alert.getId()),
                ex -> log.error("Failed to send alert: {}", ex.getMessage())
            );
    }

    public void sendProcessedAlert(Alert alert) {
        log.info("Producing processed alert: {}", alert.getId());
        kafkaTemplate.send(PROCESSED_ALERTS_TOPIC, alert.getId(), alert);
    }
}
```
## Alert Consumer Service
```
// AlertConsumer.java
@Service
@Slf4j
public class AlertConsumer {

    @Autowired
    private CorrelationService correlationService;

    @Autowired
    private AlertProducer alertProducer;

    @KafkaListener(topics = "alerts.raw", groupId = "correlation-engine")
    public void consumeRawAlert(ConsumerRecord<String, Alert> record) {
        log.info("Consuming raw alert: {}", record.key());
        
        Alert alert = record.value();
        
        try {
            // Process and correlate alert
            Alert processedAlert = correlationService.processAlert(alert);
            
            // Send to processed topic
            alertProducer.sendProcessedAlert(processedAlert);
            
            // Send WebSocket notification
            webSocketService.broadcastAlert(processedAlert);
            
        } catch (Exception e) {
            log.error("Error processing alert {}: {}", alert.getId(), e.getMessage());
            // Send to DLQ (Dead Letter Queue)
            sendToDlq(record);
        }
    }

    @KafkaListener(topics = "alerts.processed", groupId = "dashboard-service")
    public void consumeProcessedAlert(Alert alert) {
        log.info("Dashboard received processed alert: {}", alert.getId());
        // Update dashboard metrics
        metricsService.updateAlertMetrics(alert);
    }
}
```
## Integration with Correlation Service

```
// CorrelationService.java with Kafka integration
@Service
public class CorrelationService {

    @Autowired
    private AlertProducer alertProducer;

    @Autowired
    private RuleEngine ruleEngine;

    public Alert correlateAlerts(List<Alert> alerts) {
        log.info("Starting correlation of {} alerts", alerts.size());
        
        // Process each alert through Kafka
        alerts.forEach(alert -> {
            // Send to Kafka for processing
            alertProducer.sendRawAlert(alert);
            
            // Apply correlation rules
            CorrelationResult result = ruleEngine.applyRules(alert);
            
            if (result.isCorrelated()) {
                createIncident(result);
            }
        });
        
        // Stream processing for real-time correlation
        return processAlertStream(alerts);
    }

    private void createIncident(CorrelationResult result) {
        Incident incident = new Incident();
        incident.setCorrelatedAlerts(result.getAlerts());
        incident.setConfidenceScore(result.getConfidence());
        
        // Send to incidents topic
        kafkaTemplate.send("incidents.correlated", incident.getId(), incident);
        
        log.info("Created incident {} with confidence: {}", 
                 incident.getId(), result.getConfidence());
    }
}
```
