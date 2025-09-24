# Event Gateway Relay System - Configuration Guide

## Overview

The Event Gateway Relay System provides **automatic failover** for gRPC client streams across multiple clusters. When a client stream becomes inactive (5-second timeout), pending egress messages are automatically relayed to other clusters with active streams for that client.

---

## Architecture Overview

### System Components

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Cluster C1 │    │  Cluster C2 │    │  Cluster C3 │
│             │    │             │    │             │
│ Client      │    │ Client      │    │ Client      │
│ Streams     │    │ Streams     │    │ Streams     │
│             │    │             │    │             │
│ Stream      │    │ Stream      │    │ Stream      │
│ Monitor     │    │ Monitor     │    │ Monitor     │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └─────────┬─────────┼─────────┬─────────┘
                 │         │         │
           ┌─────────────────────────────┐
           │      NATS Message Bus       │
           │                             │
           │  relay.from-c1              │
           │  relay.from-c2              │
           │  relay.from-c3              │
           └─────────────────────────────┘
```

### Key Components

- **Stream Monitor**: Detects 5-second inactivity on client gRPC streams
- **Relay Publisher**: Publishes egress messages when local stream becomes inactive
- **Relay Consumer**: Receives and processes relayed messages from other clusters
- **NATS Subjects**: One relay topic per cluster for message routing

### Message Flow

#### Normal Operation
```
Client ←→ C1 Stream ←→ NATS ←→ EDA Systems
```

#### Relay Operation (Stream Inactive)
```
1. Client connects to C1 Stream [ACTIVE]
2. EDA System → NATS → egress message for Client
3. C1 Stream becomes [INACTIVE] after 5 seconds
4. C1 publishes message to relay.from-c1
5. Either C2 or C3 receives message (NATS queue group)
6. Target cluster delivers to Client via active stream
```

---

## Configuration Strategy

### Design Principle
**Explicit secondary cluster configuration** - each cluster defines which other clusters handle its relays.

### Relay Configuration Matrix

#### NATS Subject & Queue Group Mapping

| Source Cluster | Relay Subject | Queue Group | Target Consumers |
|----------------|---------------|-------------|------------------|
| **C1** | `relay.from-c1` | `qgroup-relay-from-c1` | C2, C3 instances |
| **C2** | `relay.from-c2` | `qgroup-relay-from-c2` | C1, C3 instances |
| **C3** | `relay.from-c3` | `qgroup-relay-from-c3` | C1, C2 instances |

#### Publisher/Subscriber Matrix

| Cluster | Publishes To | Subscribes To |
|---------|-------------|---------------|
| **C1** | `relay.from-c1` | `relay.from-c2`<br/>`relay.from-c3` |
| **C2** | `relay.from-c2` | `relay.from-c1`<br/>`relay.from-c3` |
| **C3** | `relay.from-c3` | `relay.from-c1`<br/>`relay.from-c2` |

---

## Complete Configuration Files

### Cluster C1 Configuration
**File:** `application-c1.properties`

```properties
# Primary cluster identity
cluster.id=c1
cluster.secondary-clusters=c2,c3

# Relay consumer configuration
nats.consumers.relay-events.enabled=true

# Subscribe to C2 relays
nats.consumers.relay-events.c2.subject=relay.from-c2
nats.consumers.relay-events.c2.queue-group=qgroup-relay-from-c2

# Subscribe to C3 relays  
nats.consumers.relay-events.c3.subject=relay.from-c3
nats.consumers.relay-events.c3.queue-group=qgroup-relay-from-c3

# Relay publisher configuration
nats.publishers.relay-publisher.subject=relay.from-c1
nats.publishers.relay-publisher.enabled=true
```

### Cluster C2 Configuration
**File:** `application-c2.properties`

```properties
# Primary cluster identity
cluster.id=c2
cluster.secondary-clusters=c1,c3

# Relay consumer configuration
nats.consumers.relay-events.enabled=true

# Subscribe to C1 relays
nats.consumers.relay-events.c1.subject=relay.from-c1
nats.consumers.relay-events.c1.queue-group=qgroup-relay-from-c1

# Subscribe to C3 relays
nats.consumers.relay-events.c3.subject=relay.from-c3
nats.consumers.relay-events.c3.queue-group=qgroup-relay-from-c3

# Relay publisher configuration
nats.publishers.relay-publisher.subject=relay.from-c2
nats.publishers.relay-publisher.enabled=true
```

### Cluster C3 Configuration  
**File:** `application-c3.properties`

```properties
# Primary cluster identity
cluster.id=c3
cluster.secondary-clusters=c1,c2

# Relay consumer configuration
nats.consumers.relay-events.enabled=true

# Subscribe to C1 relays
nats.consumers.relay-events.c1.subject=relay.from-c1
nats.consumers.relay-events.c1.queue-group=qgroup-relay-from-c1

# Subscribe to C2 relays
nats.consumers.relay-events.c2.subject=relay.from-c2
nats.consumers.relay-events.c2.queue-group=qgroup-relay-from-c2

# Relay publisher configuration
nats.publishers.relay-publisher.subject=relay.from-c3
nats.publishers.relay-publisher.enabled=true
```

---

## Configuration Pattern Summary

### Only Two Properties Change Per Cluster:
1. **`cluster.id`** - Identifies the current cluster (c1, c2, or c3)
2. **`cluster.secondary-clusters`** - Lists other clusters that can handle relays

### Everything Else Follows Pattern:
- **Consumer subjects**: `relay.from-{other-cluster-id}`
- **Queue groups**: `qgroup-relay-from-{other-cluster-id}`  
- **Publisher subject**: `relay.from-{current-cluster-id}`

### Deployment Matrix:

| Environment | Set `cluster.id` | Set `cluster.secondary-clusters` |
|-------------|------------------|----------------------------------|
| **Cluster 1** | `c1` | `c2,c3` |
| **Cluster 2** | `c2` | `c1,c3` |
| **Cluster 3** | `c3` | `c1,c2` |

---

## Implementation Architecture

### Technology Stack
- **Java 17** with **Spring Boot 3.3.9**
- **NATS** for message bus with queue groups
- **gRPC** for client streams
- **Configuration-driven** setup with explicit cluster mapping

### Core Components

#### 1. NatsConfig (Configuration Management)
```java
@ConfigurationProperties(prefix = "nats")
@Data
@Component
public class NatsConfig {
    
    @Value("${cluster.id}")
    private String clusterId;
    
    @Value("#{'${cluster.secondary-clusters}'.split(',')}")
    private List<String> secondaryClusters;
    
    // Dynamic configuration loading and validation
}
```

#### 2. ConsumerRegistrationService (Consumer Setup)
```java
@Service
public class ConsumerRegistrationService {
    
    @PostConstruct
    public void registerAllConsumers() {
        // Registers relay consumers for each secondary cluster
        // Creates NATS consumers with proper queue groups
    }
}
```

#### 3. RelayPublisherService (Message Publishing)
```java
@Service
public class RelayPublisherService {
    
    public void publishRelayFromInactiveStream(String streamId, 
                                             String clientId, 
                                             Message originalEgressMessage) {
        // Publishes to relay.from-{cluster-id} when stream inactive
    }
}
```

#### 4. RelayEventsConsumer (Message Processing)
```java
public class RelayEventsConsumer implements Consumer<Message> {
    
    @Override
    public void accept(Message natsMessage) {
        // Processes relay messages from other clusters
        // Delivers to active local streams
    }
}
```

---

## Key Design Benefits

✅ **Single Processing**: NATS queue groups ensure only one cluster processes each relay message  
✅ **Load Balancing**: Automatic distribution between available target clusters  
✅ **No Duplicates**: Each message published once to single subject  
✅ **Zero Misconfiguration**: Explicit configuration prevents routing errors  
✅ **Flexible**: Easy to add/remove clusters or customize routing rules  
✅ **Direct Message Handling**: Uses NATS Message directly with header-based metadata  
✅ **Header-based Relay Info**: Relay metadata in headers, original payload preserved

---

## Message Structure

### Original Message Flow
```
NATS Message {
    subject: "events.order.processed"
    data: [original egress payload]
    headers: {
        "event-type": "order",
        "client-id": "client-123"
    }
}
```

### Relay Message Flow
```
NATS Message {
    subject: "relay.from-c1"
    data: [same original egress payload]
    headers: {
        "relay-stream-id": "stream-456",
        "relay-client-id": "client-123", 
        "relay-original-cluster": "c1",
        "relay-timestamp": "1703123456789",
        "relay-original-subject": "events.order.processed",
        "original-event-type": "order"
    }
}
```

---

## Operational Considerations

### Monitoring Points
- **Stream Activity**: Monitor 5-second inactivity detection
- **Relay Publishing**: Track relay message publication rates
- **Consumer Processing**: Monitor relay message processing across clusters
- **Queue Group Distribution**: Verify load balancing across target clusters

### Troubleshooting
- **Check Configuration**: Verify secondary cluster lists match actual deployment
- **NATS Connectivity**: Ensure all clusters can publish/subscribe to relay subjects
- **Queue Group Membership**: Confirm consumers join correct queue groups
- **Header Validation**: Validate relay headers are properly set and read

### Scaling Considerations
- **Add New Cluster**: Update `cluster.secondary-clusters` in all existing clusters
- **Remove Cluster**: Update secondary cluster lists and restart consumers
- **Load Balancing**: NATS queue groups automatically distribute load

---

## Deployment Checklist

### Pre-Deployment
- [ ] Verify NATS connectivity between all clusters
- [ ] Confirm configuration files are environment-specific
- [ ] Test queue group functionality
- [ ] Validate subject naming conventions

### Deployment Steps
1. **Deploy Configuration**: Apply cluster-specific properties files
2. **Start Services**: Start Event Gateway services on all clusters  
3. **Verify Consumers**: Check relay consumer registration in logs
4. **Test Failover**: Simulate stream inactivity and verify relay
5. **Monitor Metrics**: Watch relay message flow and processing

### Post-Deployment Validation
- [ ] Stream inactivity detection working (5-second timeout)
- [ ] Relay messages publishing to correct subjects
- [ ] Target clusters receiving and processing relays
- [ ] No duplicate message processing
- [ ] Client messages delivered successfully via any cluster

This design ensures **reliable failover** with **zero duplicate processing** across the three-cluster deployment.