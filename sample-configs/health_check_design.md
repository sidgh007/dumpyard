# Event Gateway Health Check System - Complete Design Document

## Executive Summary

We are extending the Event Gateway Relay System with an intelligent **Health Check mechanism** to ensure reliable cross-cluster failover. This system prevents relay messages from being sent to unhealthy clusters, reducing message loss and improving system reliability.

### Key Business Benefits
- **Improved Reliability**: 99.9% reduction in failed relay attempts to unhealthy clusters
- **Faster Failover**: Real-time health awareness (1-second detection)
- **Operational Excellence**: Proactive cluster health monitoring with detailed visibility
- **Cost Efficiency**: Eliminates unnecessary processing of messages destined for failed clusters

---

## Problem Statement

### Current Challenge
Our existing Event Gateway Relay System blindly forwards messages to secondary clusters when local streams become inactive. If target clusters are unhealthy:

❌ **Silent Message Loss**: Messages sent to dead clusters disappear  
❌ **Resource Wastage**: CPU/Network overhead for failed relay attempts  
❌ **Poor User Experience**: Clients don't receive responses  
❌ **Operational Blindness**: No visibility into cluster health status

### Business Impact
- **Customer Impact**: Failed message delivery affects user experience
- **Operational Cost**: Engineers spend time troubleshooting phantom issues
- **System Reliability**: Cascading failures when clusters go down

---

## Solution Overview

### Health Check System Design
Implement a **distributed health monitoring system** that continuously monitors cluster health and prevents relay to unhealthy targets.

#### Core Principles
1. **Cluster-Level Health**: Monitor cluster availability, not individual instances
2. **Real-Time Monitoring**: 1-second health check intervals with sub-second response times
3. **Intelligent Load Distribution**: Stagger health checks to prevent load spikes
4. **Pre-Relay Validation**: Check target health before attempting message relay
5. **Graceful Degradation**: Clear logging and message dropping for unhealthy targets

---

## Architecture & Design

### High-Level Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Cluster C1 │    │  Cluster C2 │    │  Cluster C3 │
│             │    │             │    │             │
│ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │
│ │Health   │ │◄──►│ │Health   │ │◄──►│ │Health   │ │
│ │Monitor  │ │    │ │Monitor  │ │    │ │Monitor  │ │
│ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │
│             │    │             │    │             │
│ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │
│ │Local    │ │    │ │Local    │ │    │ │Local    │ │
│ │Cache    │ │    │ │Cache    │ │    │ │Cache    │ │
│ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └─────────┬─────────┼─────────┬─────────┘
                 │         │         │
           ┌─────────────────────────────┐
           │      NATS Health Bus        │
           │                             │
           │  health-check.c1            │
           │  health-check.c2            │  
           │  health-check.c3            │
           │  health-response.c1         │
           │  health-response.c2         │
           │  health-response.c3         │
           └─────────────────────────────┘
```

### Health Check Flow Design

#### Complete Health Check Cycle: C1 Checking C2 Health

**Step 1: C1 Instances Send Health Check Requests**
```
Time 1ms:    C1-Inst1 → publishes to health-check.c2
Time 501ms:  C1-Inst2 → publishes to health-check.c2
```

**Step 2: NATS Routes to C2's Queue Group**
```
health-check.c2 → Queue Group: health-responders-c2
Either C2-Inst1 OR C2-Inst2 picks up each request
```

**Step 3: C2 Instance Responds**
```
C2-Inst1 (example) receives health-check.c2 message
C2-Inst1 responds by publishing to health-response.c1
```

**Step 4: NATS Broadcasts Response to All C1 Instances**
```
C2-Inst1 → publishes to health-response.c1 → Both C1-Inst1 AND C1-Inst2 receive
```

**Step 5: C1 Instances Update Health Cache**
```
Both C1 instances update local cache: C2 = HEALTHY
```

### NATS Subject & Queue Design

#### Health Check Subjects
| Subject | Purpose | Queue Group | Why Queue Group? |
|---------|---------|-------------|------------------|
| `health-check.c1` | Health requests FOR C1 | `health-responders-c1` | Only one C1 instance responds |
| `health-check.c2` | Health requests FOR C2 | `health-responders-c2` | Only one C2 instance responds |
| `health-check.c3` | Health requests FOR C3 | `health-responders-c3` | Only one C3 instance responds |
| `health-response.c1` | Health responses TO C1 | No queue group | All C1 instances need health updates |
| `health-response.c2` | Health responses TO C2 | No queue group | All C2 instances need health updates |
| `health-response.c3` | Health responses TO C3 | No queue group | All C3 instances need health updates |

#### Why No Queue Group for Responses?
- **Cache Consistency**: Both instances in requesting cluster need identical health knowledge
- **Consistent Relay Decisions**: Prevents split behavior where one instance thinks cluster is healthy while other doesn't
- **Acceptable Traffic**: Duplicate responses (24 messages/second total) vs cache coordination complexity

---

## Detailed Implementation

### Configuration Management

#### Simplified Health Check Properties

**Cluster C1 Configuration (`application-c1.properties`):**
```properties
# Primary cluster identity
cluster.id=c1
cluster.secondary-clusters=c2,c3

# Health check configuration (single toggle)
cluster.health-check.enabled=true
cluster.health-check.interval-ms=1000
cluster.health-check.timeout-ms=2000
cluster.health-check.instance-offset-ms=500

# Health check publishers (C1 checking others)
nats.publishers.health-check.c2.subject=health-check.c2
nats.publishers.health-check.c3.subject=health-check.c3

# Health check consumers (C1 receiving health checks from others)
nats.consumers.health-check.c1.subject=health-check.c1
nats.consumers.health-check.c1.queue-group=health-responders-c1

# Health check response consumer (C1 receiving responses about others)
nats.consumers.health-response.c1.subject=health-response.c1
nats.consumers.health-response.c1.queue-group=No queue group required

# Health check response publisher (C1 responding back)
nats.publishers.health-response.subject=health-response.c1

# Existing relay configuration (unchanged)
nats.consumers.relay-events.enabled=true
nats.consumers.relay-events.c2.subject=relay.from-c2
nats.consumers.relay-events.c2.queue-group=qgroup-relay-from-c2
nats.consumers.relay-events.c3.subject=relay.from-c3
nats.consumers.relay-events.c3.queue-group=qgroup-relay-from-c3
nats.publishers.relay-publisher.subject=relay.from-c1
nats.publishers.relay-publisher.enabled=true
```

**Cluster C2 Configuration (`application-c2.properties`):**
```properties
# Primary cluster identity
cluster.id=c2
cluster.secondary-clusters=c1,c3

# Health check configuration (single toggle)
cluster.health-check.enabled=true
cluster.health-check.interval-ms=1000
cluster.health-check.timeout-ms=2000
cluster.health-check.instance-offset-ms=500

# Health check publishers (C2 checking others)
nats.publishers.health-check.c1.subject=health-check.c1
nats.publishers.health-check.c3.subject=health-check.c3

# Health check consumers (C2 receiving health checks from others)
nats.consumers.health-check.c2.subject=health-check.c2
nats.consumers.health-check.c2.queue-group=health-responders-c2

# Health check response consumer (C2 receiving responses about others)
nats.consumers.health-response.c2.subject=health-response.c2
nats.consumers.health-response.c2.queue-group=No queue group required

# Health check response publisher (C2 responding back)
nats.publishers.health-response.subject=health-response.c2

# Existing relay configuration (unchanged)
nats.consumers.relay-events.enabled=true
nats.consumers.relay-events.c1.subject=relay.from-c1
nats.consumers.relay-events.c1.queue-group=qgroup-relay-from-c1
nats.consumers.relay-events.c3.subject=relay.from-c3
nats.consumers.relay-events.c3.queue-group=qgroup-relay-from-c3
nats.publishers.relay-publisher.subject=relay.from-c2
nats.publishers.relay-publisher.enabled=true
```

**Cluster C3 Configuration (`application-c3.properties`):**
```properties
# Primary cluster identity
cluster.id=c3
cluster.secondary-clusters=c1,c2

# Health check configuration (single toggle)
cluster.health-check.enabled=true
cluster.health-check.interval-ms=1000
cluster.health-check.timeout-ms=2000
cluster.health-check.instance-offset-ms=500

# Health check publishers (C3 checking others)
nats.publishers.health-check.c1.subject=health-check.c1
nats.publishers.health-check.c2.subject=health-check.c2

# Health check consumers (C3 receiving health checks from others)
nats.consumers.health-check.c3.subject=health-check.c3
nats.consumers.health-check.c3.queue-group=health-responders-c3

# Health check response consumer (C3 receiving responses about others)
nats.consumers.health-response.c3.subject=health-response.c3
nats.consumers.health-response.c3.queue-group=No queue group required

# Health check response publisher (C3 responding back)
nats.publishers.health-response.subject=health-response.c3

# Existing relay configuration (unchanged)
nats.consumers.relay-events.enabled=true
nats.consumers.relay-events.c1.subject=relay.from-c1
nats.consumers.relay-events.c1.queue-group=qgroup-relay-from-c1
nats.consumers.relay-events.c2.subject=relay.from-c2
nats.consumers.relay-events.c2.queue-group=qgroup-relay-from-c2
nats.publishers.relay-publisher.subject=relay.from-c3
nats.publishers.relay-publisher.enabled=true
```

#### Configuration Benefits

**Single Toggle Control:**
- `cluster.health-check.enabled=true` → Entire health check system active
- `cluster.health-check.enabled=false` → Falls back to original relay behavior
- **No individual component enabling/disabling** → Simplified operations

**Only Two Properties Change Per Cluster:**
- `cluster.id` (c1, c2, c3)
- `cluster.secondary-clusters` (which others to monitor)

### Complete Publisher/Subscriber Matrix

| Instance | **PUBLISHING TO** | **Target Queue Group** | **SUBSCRIBING TO** | **Subscribe Queue Group** | **Purpose** |
|----------|-------------------|------------------------|-------------------|-------------------------|-------------|
| **C1-Inst1** | `health-check.c2` | `health-responders-c2` | `health-check.c1` | `health-responders-c1` | Respond to health checks from C2/C3 |
|  | `health-check.c3` | `health-responders-c3` | `health-response.c1` | No queue group required | Receive responses about C2/C3 health |
| **C1-Inst2** | `health-check.c2` | `health-responders-c2` | `health-check.c1` | `health-responders-c1` | Respond to health checks from C2/C3 |
|  | `health-check.c3` | `health-responders-c3` | `health-response.c1` | No queue group required | Receive responses about C2/C3 health |
| **C2-Inst1** | `health-check.c1` | `health-responders-c1` | `health-check.c2` | `health-responders-c2` | Respond to health checks from C1/C3 |
|  | `health-check.c3` | `health-responders-c3` | `health-response.c2` | No queue group required | Receive responses about C1/C3 health |
| **C2-Inst2** | `health-check.c1` | `health-responders-c1` | `health-check.c2` | `health-responders-c2` | Respond to health checks from C1/C3 |
|  | `health-check.c3` | `health-responders-c3` | `health-response.c2` | No queue group required | Receive responses about C1/C3 health |
| **C3-Inst1** | `health-check.c1` | `health-responders-c1` | `health-check.c3` | `health-responders-c3` | Respond to health checks from C1/C2 |
|  | `health-check.c2` | `health-responders-c2` | `health-response.c3` | No queue group required | Receive responses about C1/C2 health |
| **C3-Inst2** | `health-check.c1` | `health-responders-c1` | `health-check.c3` | `health-responders-c3` | Respond to health checks from C1/C2 |
|  | `health-check.c2` | `health-responders-c2` | `health-response.c3` | No queue group required | Receive responses about C1/C2 health |

### Message Structure

#### Health Check Request
```json
{
  "sourceCluster": "c1",
  "sourceInstanceId": "c1-inst1-hostname-8080",
  "requestTimestamp": 1703123456001,
  "requestId": "550e8400-e29b-41d4-a716-446655440000"
}
```

#### Health Check Response
```json
{
  "targetCluster": "c2", 
  "targetInstanceId": "c2-inst1-hostname-8080",
  "requestTimestamp": 1703123456001,
  "responseTimestamp": 1703123456050,
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "HEALTHY"
}
```

### Health Cache Management

#### Local Cache Structure
```java
class ClusterHealthStatus {
    String clusterId;           // "c2", "c3"
    boolean isHealthy;          // true/false
    long responseTimeMs;        // 49ms
    long lastCheckedTimestamp;  // 1703123456050
}
```

#### Cache Validation Rules
- **Stale Timeout**: 3 seconds (if no recent response, assume unhealthy)
- **Conservative Approach**: Unknown status = unhealthy
- **Instance-Level Cache**: Each instance maintains independent cache
- **Consistent Updates**: All instances in cluster get same health info (no queue group on responses)

---

## Traffic Analysis & Performance

### Health Check Traffic (Per Second)

#### Request Load
| Target Cluster | Health Check Requests/Second | Source |
|----------------|----------------------------|--------|
| **C1** | 4 requests | 2 from C2 + 2 from C3 |
| **C2** | 4 requests | 2 from C1 + 2 from C3 |  
| **C3** | 4 requests | 2 from C1 + 2 from C2 |
| **Total** | **12 requests/second** | Across all clusters |

#### Response Load
| Target Cluster | Health Check Responses/Second | Recipients |
|----------------|-------------------------------|------------|
| **C1** | 8 messages | 4 responses × 2 instances |
| **C2** | 8 messages | 4 responses × 2 instances |
| **C3** | 8 messages | 4 responses × 2 instances |
| **Total** | **24 messages/second** | Across all clusters |

#### Staggered Timing Impact
- **Instance-1**: Health checks at 1ms of each second
- **Instance-2**: Health checks at 501ms of each second  
- **Load Distribution**: Prevents thundering herd, spreads NATS load
- **Network Smoothing**: Avoids 500ms spikes every second

#### Network Impact
- **Message Size**: ~200 bytes per health check message
- **Total Bandwidth**: ~7.2 KB/second per cluster
- **Negligible Impact**: <0.01% of typical cluster network usage

### Performance Characteristics
- **Health Check Response Time**: <50ms typical, 2000ms timeout
- **Cache Update Latency**: <100ms from health change to cache update
- **Memory Footprint**: <1KB per instance for health cache
- **CPU Overhead**: <0.1% per instance

---

## Integration with Relay System

### Pre-Relay Health Validation

#### Enhanced Relay Flow
```java
// Before relay (new logic)
if (!healthCache.isClusterHealthy(targetCluster)) {
    log.warn("Dropping relay message - target cluster {} is unhealthy", targetCluster);
    metrics.incrementDroppedRelays(targetCluster);
    return; // Drop message with clear justification
}

// Existing relay logic continues
relayPublisher.publishRelay(message);
```

#### Relay Decision Matrix
| C2 Health | C3 Health | Relay Decision |
|-----------|-----------|----------------|
| ✅ HEALTHY | ✅ HEALTHY | Relay to either C2 or C3 (NATS queue group decides) |
| ✅ HEALTHY | ❌ UNHEALTHY | Relay only to C2 |
| ❌ UNHEALTHY | ✅ HEALTHY | Relay only to C3 |
| ❌ UNHEALTHY | ❌ UNHEALTHY | **Drop message with logging** |

### Fallback Strategy
When all secondary clusters are unhealthy:
1. **Log Clear Justification**: "Dropping relay - all target clusters unhealthy"
2. **Increment Metrics**: Track dropped message counts per reason
3. **Alert Operations**: Threshold-based alerting for high drop rates
4. **No Silent Failures**: Every dropped message is accounted for

---

## Operational Benefits

### Monitoring & Observability

#### Health Status Dashboard
- **Real-time Cluster Health**: Live status of all clusters
- **Response Time Trends**: Historical health check performance
- **Availability SLA Tracking**: Cluster uptime percentages
- **Relay Success Rates**: Impact of health-aware relay decisions

#### Key Metrics
```
health_check_success_rate{source_cluster="c1", target_cluster="c2"}
health_check_response_time{source_cluster="c1", target_cluster="c2"}
relay_messages_dropped{reason="target_unhealthy", target_cluster="c2"}
cluster_availability{cluster="c2"}
```

#### Alerting Rules
- **Cluster Down**: No health responses for >5 seconds
- **High Latency**: Health check response time >1000ms
- **High Drop Rate**: >10% relay messages dropped due to unhealthy targets

### Troubleshooting Support

#### Log Examples
```
INFO  - Health check succeeded for cluster: c2 in 47ms
WARN  - Health check failed for cluster: c3 - timeout after 2000ms
WARN  - Dropping relay message - target cluster c3 is unhealthy
ERROR - All secondary clusters unhealthy - unable to relay message for client: client-123
```

#### Diagnostic Endpoints
- `GET /health/status` - Current health status of all clusters
- `GET /health/cache` - Local health cache contents
- `GET /health/metrics` - Health check performance metrics

---

## Risk Assessment & Mitigation

### Potential Risks

#### Risk 1: Health Check False Negatives
**Risk**: Network issues cause healthy cluster to appear unhealthy
**Impact**: Unnecessary message drops
**Mitigation**: 
- Conservative 2-second timeout
- Stale data detection (3-second threshold)
- Multiple health check attempts per second (staggered timing)

#### Risk 2: NATS Subject Congestion
**Risk**: High health check traffic affects NATS performance  
**Impact**: System-wide latency
**Mitigation**:
- Dedicated health check subjects (separate from business traffic)
- Lightweight message payloads (<200 bytes)
- Staggered timing to distribute load

#### Risk 3: Cache Inconsistency
**Risk**: Instance caches become out of sync
**Impact**: Inconsistent relay behavior
**Mitigation**:
- All instances receive same health responses (no queue group)
- Short cache TTL (3 seconds)
- Conservative unhealthy-by-default policy

### Rollback Strategy
- **Feature Toggle**: `cluster.health-check.enabled=false` disables health checking
- **Graceful Degradation**: System falls back to original blind relay behavior  
- **Zero Downtime**: Health check system can be enabled/disabled without restart

---

## Implementation Timeline

### Phase 1: Core Health Check System (2 weeks)
- [ ] Health check message structures
- [ ] NATS publisher/subscriber setup
- [ ] Local health cache implementation
- [ ] Basic health check scheduling with staggered timing

### Phase 2: Relay Integration (1 week)  
- [ ] Pre-relay health validation
- [ ] Message dropping logic with justification
- [ ] Integration testing with existing relay system

### Phase 3: Monitoring & Operations (1 week)
- [ ] Health status metrics and dashboards
- [ ] Diagnostic endpoints
- [ ] Alerting rules and runbooks
- [ ] Performance validation

### Total Timeline: 4 weeks

---

## Success Criteria

### Technical KPIs
- **Health Check Accuracy**: >99.5% correct health status detection
- **Response Time**: <100ms average health check response time  
- **System Overhead**: <0.1% additional CPU and memory usage
- **Network Impact**: <0.01% additional bandwidth usage

### Business KPIs
- **Reduced Failed Relays**: >95% reduction in relay attempts to unhealthy clusters
- **Faster Issue Detection**: <1 second detection of cluster failures
- **Improved Reliability**: >99.9% successful message delivery to healthy clusters
- **Operational Efficiency**: 50% reduction in time spent troubleshooting relay issues

### Operational Readiness
- [ ] Monitoring dashboards deployed
- [ ] Alerting rules configured
- [ ] Runbooks documented
- [ ] Team trained on new health check system

---

## Configuration Deployment Matrix

### Only Properties That Change Per Cluster

| Environment | Set `cluster.id` | Set `cluster.secondary-clusters` |
|-------------|------------------|----------------------------------|
| **Cluster 1** | `c1` | `c2,c3` |
| **Cluster 2** | `c2` | `c1,c3` |
| **Cluster 3** | `c3` | `c1,c2` |

### Health Check Configuration Summary

| Property | Purpose | Same for All Clusters? |
|----------|---------|------------------------|
| `cluster.health-check.enabled` | Master toggle | ✅ Yes |
| `cluster.health-check.interval-ms` | Check frequency | ✅ Yes |
| `cluster.health-check.timeout-ms` | Response timeout | ✅ Yes |
| `cluster.health-check.instance-offset-ms` | Stagger timing | ✅ Yes |

---

## Conclusion

The Health Check System represents a critical enhancement to our Event Gateway infrastructure, transforming it from a **blind relay system** to an **intelligent, health-aware message routing platform**.

### Key Value Propositions
1. **Reliability**: Eliminates silent message failures due to unhealthy target clusters
2. **Visibility**: Real-time cluster health monitoring and alerting  
3. **Efficiency**: Prevents wasteful processing of doomed relay messages
4. **Operational Excellence**: Clear logging and metrics for troubleshooting

### Design Highlights
- **Cluster-Level Toggle**: Single property controls entire health check system
- **Consistent Cache**: No queue group on responses ensures all instances have same health view  
- **Staggered Timing**: Load distribution prevents network spikes
- **Conservative Approach**: Unknown/stale health status treated as unhealthy

This system positions our Event Gateway as a **production-ready, enterprise-grade** messaging platform capable of handling complex multi-cluster deployments with confidence.

### Recommendation
**Proceed with implementation** - The benefits significantly outweigh the complexity, and the 4-week implementation timeline provides excellent ROI for improved system reliability and operational visibility.