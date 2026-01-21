# Directory Structure
```

le-alert-correlation-system/
├── 📁 kafka-config/
│   ├── docker-compose.kafka.yml
│   ├── kafka-setup.sh
│   ├── create-topics.sh
│   └── schema-registry/
│       ├── alert.avsc
│       └── incident.avsc
├── 📁 backend/
│   ├── src/main/java/com/le/correlation/kafka/
│   │   ├── AlertProducer.java
│   │   ├── AlertConsumer.java
│   │   ├── IncidentProducer.java
│   │   ├── KafkaConfig.java
│   │   └── KafkaTopicConfig.java
│   └── resources/application-kafka.yml
└── 📁 frontend/
    └── src/services/kafka/
        ├── kafkaClient.ts
        └── alertConsumer.ts

```
