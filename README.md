# KAFKA
Kafka Integration Setup &amp; Implementation Guide for Law Enforcement - LE.

##  Data Flow Architecture

```
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│                     │     │                     │     │                     │
│    Data Sources     │────▶│   Ingest Service    │────▶│   Kafka Cluster     │
│   (911, Sensors,    │     │   (REST API /       │     │    ┌─────────────┐  │
│    Social Media)    │     │    WebSocket)       │     │    │ alerts.raw  │  │
│                     │     │                     │     │    │   Topic     │  │
└─────────────────────┘     └─────────────────────┘     │    └─────────────┘  │
         │                           │                  │           │         │
         │                           │                  │           ▼         │
         │                           │                  │    ┌─────────────┐  │
         │                           │                  │    │Correlation  │  │
         │                           │                  │    │  Engine     │  │
         │                           │                  │    │ (Quarkus/   │  │
         │                           │                  │    │   Spring)   │  │
         │                           │                  │    └─────────────┘  │
         │                           │                  │           │         │
         │                           │                  │           ▼         │
         │                           │                  │    ┌─────────────┐  │
         │                           │                  │    │alerts.      │  │
         │                           │                  │    │ processed   │  │
┌─────────────────────┐              │                  │    │   Topic     │  │
│                     │              │                  │    └─────────────┘  │
│    Frontend UI      │◀────────────-┼──────────────────┤           │         │
│   (React/TypeScript)│              │                  │           ▼         │
│                     │              │                  │    ┌─────────────┐  │
└─────────────────────┘              │                  │    │incidents.   │  │
         │                           │                  │    │ correlated  │  │
         │                           │                  │    │   Topic     │  │
         │                           │                  │    └─────────────┘  │
         ▼                           ▼                  └──────────┬─────────-┘
┌─────────────────────┐     ┌─────────────────────┐                │
│                     │     │                     │                │
│   Kafka Consumer    │◀────┤   Kafka Streams /   │◀──────────────-┘
│   (Frontend via     │     │   Spring Consumers  │
│    WebSocket Proxy) │     │                     │
│                     │     └─────────────────────┘
└─────────────────────┘               │
         │                            │
         ▼                            ▼
┌─────────────────────┐     ┌─────────────────────┐
│                     │     │                     │
│  React Context &    │     │  Dashboard Service  │
│     State Update    │     │  (Metrics, Analytics│
│                     │     │     & Reporting)    │
└─────────────────────┘     └─────────────────────┘
         │                            │
         ▼                            ▼
┌─────────────────────┐     ┌─────────────────────┐
│                     │     │                     │
│   UI Re-render &    │     │   Real-time         │
│   Component Update  │     │   Notifications     │
│                     │     │   (Push, Email,     │
└─────────────────────┘     │      SMS)           │
                             └─────────────────────┘
```
## Step-by-Step Sequence

- Alert Ingestion
  - Source generates alert (911 call, sensor, etc.)
  - Alert sent to /api/alerts endpoint
  - Controller publishes to alerts.raw topic

- Kafka Processing
  - Correlation Engine consumes from alerts.raw
  - Processes alert through RuleEngine
  - Publishes processed alert to alerts.processed
  - If correlation detected, creates incident in incidents.correlated

- Frontend Consumption
  - Frontend Kafka consumer subscribes to alerts.processed
  - Real-time updates via React hooks
  - State updates in AlertContext
  - UI re-renders with new alerts

- Fallback Mechanism
  - If Kafka fails, fallback to REST API
  - WebSocket backup for real-time updates
  - Local storage for offline capability

## Prerequisites and Dependencies
- https://github.com/kukuu/KAFKA/blob/main/setup-and-dependencies.md

## Directory Structure
- https://github.com/kukuu/KAFKA/blob/main/directory-structure.md

## Configuration Setup
- https://github.com/kukuu/KAFKA/blob/main/configuration-setup.md
