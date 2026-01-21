# Setup & Dependencies

**1.1 Prerequisites Installation**
```
# Install Kafka & Zookeeper
brew install kafka                    # macOS
# OR
sudo apt-get install kafka            # Ubuntu
# OR using Docker
docker-compose up -d kafka zookeeper
```
**1.2 Dependencies**

- Backend Dependencies (pom.xml)

```
<dependencies>
    <!-- Spring Kafka -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
        <version>3.0.0</version>
    </dependency>
    
    <!-- Kafka Streams -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka-streams</artifactId>
    </dependency>
    
    <!-- Avro Schema Registry (Optional) -->
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>7.3.0</version>
    </dependency>
</dependencies>
```
- Frontend Dependencies (package.json)
```
{
  "dependencies": {
    "kafkajs": "^2.2.0",
    "socket.io-client": "^4.5.0",
    "rxjs": "^7.5.0"
  }
}
```
