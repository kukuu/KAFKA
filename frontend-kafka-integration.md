# Frontend Kafka Integration

## Kafka Client Configuration

```
// frontend/src/services/kafka/kafkaClient.ts
import { Kafka, logLevel } from 'kafkajs';

class KafkaClient {
  private kafka: Kafka;
  private consumer: any;
  private producer: any;
  
  constructor() {
    this.kafka = new Kafka({
      clientId: 'le-alert-frontend',
      brokers: [import.meta.env.VITE_KAFKA_BROKER || 'localhost:9092'],
      logLevel: logLevel.ERROR,
      ssl: process.env.NODE_ENV === 'production',
      sasl: process.env.NODE_ENV === 'production' ? {
        mechanism: 'plain',
        username: import.meta.env.VITE_KAFKA_USERNAME,
        password: import.meta.env.VITE_KAFKA_PASSWORD
      } : undefined
    });
    
    this.consumer = this.kafka.consumer({ groupId: 'frontend-consumer' });
    this.producer = this.kafka.producer();
  }

  async connect() {
    await this.consumer.connect();
    await this.producer.connect();
    console.log('Kafka client connected');
  }

  async subscribeToAlerts(callback: (alert: any) => void) {
    await this.consumer.subscribe({ 
      topic: 'alerts.processed', 
      fromBeginning: false 
    });

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }: any) => {
        const alert = JSON.parse(message.value.toString());
        callback(alert);
      },
    });
  }

  async sendAlert(alert: Alert) {
    await this.producer.send({
      topic: 'alerts.raw',
      messages: [
        {
          key: alert.id,
          value: JSON.stringify(alert)
        }
      ]
    });
  }

  async disconnect() {
    await this.consumer.disconnect();
    await this.producer.disconnect();
  }
}

export const kafkaClient = new KafkaClient();
```

## React Hook for Kafka Integration
```
// frontend/src/hooks/useKafkaAlerts.ts
import { useEffect, useState } from 'react';
import { kafkaClient } from '../services/kafka/kafkaClient';
import { Alert } from '../types';

export const useKafkaAlerts = () => {
  const [alerts, setAlerts] = useState<Alert[]>([]);
  const [isConnected, setIsConnected] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const connectToKafka = async () => {
      try {
        await kafkaClient.connect();
        setIsConnected(true);
        
        await kafkaClient.subscribeToAlerts((alert: Alert) => {
          setAlerts(prev => [alert, ...prev.slice(0, 49)]); // Keep last 50 alerts
        });
        
      } catch (err) {
        setError('Failed to connect to Kafka');
        console.error('Kafka connection error:', err);
      }
    };

    connectToKafka();

    return () => {
      kafkaClient.disconnect();
    };
  }, []);

  const sendAlert = async (alert: Alert) => {
    if (!isConnected) {
      throw new Error('Kafka not connected');
    }
    
    try {
      await kafkaClient.sendAlert(alert);
      return true;
    } catch (err) {
      console.error('Failed to send alert to Kafka:', err);
      return false;
    }
  };

  return { alerts, isConnected, error, sendAlert };
};
```

## Integration with Alert Context

```
// frontend/src/contexts/AlertContext.tsx (updated)
import React, { createContext, useContext, useEffect } from 'react';
import { useKafkaAlerts } from '../hooks/useKafkaAlerts';
import { Alert } from '../types';

interface AlertContextType {
  alerts: Alert[];
  addAlert: (alert: Alert) => Promise<boolean>;
  clearAlerts: () => void;
  isKafkaConnected: boolean;
}

const AlertContext = createContext<AlertContextType | undefined>(undefined);

export const AlertProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { alerts, isConnected, sendAlert } = useKafkaAlerts();
  const [localAlerts, setLocalAlerts] = useState<Alert[]>([]);

  const addAlert = async (alert: Alert): Promise<boolean> => {
    // Send to Kafka
    const kafkaSuccess = await sendAlert(alert);
    
    if (kafkaSuccess) {
      // Optimistic update
      setLocalAlerts(prev => [alert, ...prev]);
      return true;
    }
    
    // Fallback to REST API if Kafka fails
    try {
      const response = await fetch('/api/alerts', {
        method: 'POST',
        body: JSON.stringify(alert)
      });
      
      if (response.ok) {
        setLocalAlerts(prev => [alert, ...prev]);
        return true;
      }
    } catch (error) {
      console.error('Both Kafka and REST API failed:', error);
    }
    
    return false;
  };

  useEffect(() => {
    // Merge Kafka alerts with local state
    setLocalAlerts(prev => [...alerts, ...prev].slice(0, 50));
  }, [alerts]);

  return (
    <AlertContext.Provider value={{
      alerts: localAlerts,
      addAlert,
      clearAlerts: () => setLocalAlerts([]),
      isKafkaConnected: isConnected
    }}>
      {children}
    </AlertContext.Provider>
  );
};
```

