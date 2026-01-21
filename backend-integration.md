# Backend Integration

**Spring Kafka Configuration**
@Configuration: 

@EnableKafka: 


```
// KafkaConfig.java
@Configuration //Marks this as a Spring configuration class
@EnableKafka //Enables Spring's Kafka support framework
//Purpose: Both above creates a centralized configuration for Kafka messaging in the application.

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
**Topic Configuration**

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
