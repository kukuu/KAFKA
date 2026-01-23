# Deployment and Scaling

## Kubernetes Deployment
```
# kafka-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: correlation-engine
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: correlation-engine
        image: le-correlation-engine:latest
        env:
        - name: KAFKA_BROKERS
          value: "kafka-cluster:9092"
        - name: SPRING_PROFILES_ACTIVE
          value: "kafka,kubernetes"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```
## Horizontal Scaling Strategy
- Consumers: Scale based on topic partitions

- Producers: Scale based on ingress load

- Kafka Cluster: 3 brokers minimum for production

- Monitoring: Alert on consumer lag > threshold
