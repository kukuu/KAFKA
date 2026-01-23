# Debug Commands

```
# Check topic status
kafka-topics --bootstrap-server localhost:9092 --list
```
```
# View messages in topic
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic alerts.raw --from-beginning
```
```
# Check consumer groups
kafka-consumer-groups --bootstrap-server localhost:9092 --list
```
```
# Describe consumer lag
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group correlation-engine --describe
```
