# Testing Strategy

##  Backend Kafka Testing
```
// KafkaIntegrationTest.java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"alerts.raw", "alerts.processed"})
class KafkaIntegrationTest {

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Autowired
    private AlertProducer alertProducer;

    @Autowired
    private KafkaTemplate<String, Alert> kafkaTemplate;

    @Test
    void testAlertProducedAndConsumed() throws Exception {
        // Given
        Alert alert = createTestAlert();
        
        // When
        alertProducer.sendRawAlert(alert);
        
        // Then
        ConsumerRecord<String, Alert> record = KafkaTestUtils
            .getSingleRecord(kafkaTemplate, "alerts.raw", 5000);
        
        assertNotNull(record);
        assertEquals(alert.getId(), record.key());
        assertEquals(alert.getPriority(), record.value().getPriority());
    }

    @Test
    void testEndToEndFlow() {
        // Given
        Alert alert = createTestAlert();
        
        // When
        correlationService.correlateAlerts(List.of(alert));
        
        // Then - Check processed topic
        ConsumerRecord<String, Alert> processedRecord = KafkaTestUtils
            .getSingleRecord(kafkaTemplate, "alerts.processed", 10000);
        
        assertNotNull(processedRecord);
        assertTrue(processedRecord.value().isProcessed());
    }
}
```

## Frontend Kafka Testing
```
// kafkaClient.test.ts
import { kafkaClient } from './kafkaClient';
import { Kafka } from 'kafkajs';

jest.mock('kafkajs');

describe('KafkaClient', () => {
  let mockConsumer: any;
  let mockProducer: any;

  beforeEach(() => {
    mockConsumer = {
      connect: jest.fn(),
      subscribe: jest.fn(),
      run: jest.fn(),
      disconnect: jest.fn()
    };

    mockProducer = {
      connect: jest.fn(),
      send: jest.fn(),
      disconnect: jest.fn()
    };

    (Kafka as jest.Mock).mockImplementation(() => ({
      consumer: () => mockConsumer,
      producer: () => mockProducer
    }));
  });

  test('should connect to Kafka on initialization', async () => {
    await kafkaClient.connect();
    
    expect(mockConsumer.connect).toHaveBeenCalled();
    expect(mockProducer.connect).toHaveBeenCalled();
  });

  test('should send alert to Kafka', async () => {
    const alert = {
      id: 'test-123',
      priority: 'HIGH',
      location: 'Downtown'
    };

    await kafkaClient.sendAlert(alert);
    
    expect(mockProducer.send).toHaveBeenCalledWith({
      topic: 'alerts.raw',
      messages: [{
        key: 'test-123',
        value: JSON.stringify(alert)
      }]
    });
  });
});
```

## Integration Testing
```
// KafkaIntegrationTestWithContainers.java
@Testcontainers
@SpringBootTest
class KafkaIntegrationTestWithContainers {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.3.0")
    );

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Test
    void testRealKafkaIntegration() {
        // Test with actual Kafka container
        // Similar to embedded tests but with real Kafka
    }
}
```

## E2E Testing with Cypress
