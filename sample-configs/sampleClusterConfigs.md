

# Relay consumers (subscribe to other clusters)
nats.consumers.relay-events.enabled=true
nats.consumers.relay-events.c2.subject=relay.from-c2
nats.consumers.relay-events.c2.queue-group=qgroup-relay-from-c2
nats.consumers.relay-events.c3.subject=relay.from-c3
nats.consumers.relay-events.c3.queue-group=qgroup-relay-from-c3

# Relay publisher (publish own relays)
nats.publishers.relay-publisher.subject=relay.from-c1
nats.publishers.relay-publisher.enabled=true
Cluster C2 Properties
properties# Primary cluster
cluster.id=c2
cluster.secondary-clusters=c1,c3

# Relay consumers (subscribe to other clusters)
nats.consumers.relay-events.enabled=true
nats.consumers.relay-events.c1.subject=relay.from-c1
nats.consumers.relay-events.c1.queue-group=qgroup-relay-from-c1
nats.consumers.relay-events.c3.subject=relay.from-c3
nats.consumers.relay-events.c3.queue-group=qgroup-relay-from-c3

# Relay publisher (publish own relays)
nats.publishers.relay-publisher.subject=relay.from-c2
nats.publishers.relay-publisher.enabled=true
Cluster C3 Properties
properties# Primary cluster
cluster.id=c3
cluster.secondary-clusters=c1,c2

# Relay consumers (subscribe to other clusters)
nats.consumers.relay-events.enabled=true
nats.consumers.relay-events.c1.subject=relay.from-c1
nats.consumers.relay-events.c1.queue-group=qgroup-relay-from-c1
nats.consumers.relay-events.c2.subject=relay.from-c2
nats.consumers.relay-events.c2.queue-group=qgroup-relay-from-c2

# Relay publisher (publish own relays)
nats.publishers.relay-publisher.subject=relay.from-c3
nats.publishers.relay-publisher.enabled=true
NATS Subject & Queue Group Design
Source ClusterNATS SubjectQueue GroupCompeting ConsumersC1relay.from-c1qgroup-relay-from-c1C2, C3 instancesC2relay.from-c2qgroup-relay-from-c2C1, C3 instancesC3relay.from-c3qgroup-relay-from-c3C1, C2 instances
Key Design Benefits
✅ Single Processing: NATS queue groups ensure only one cluster processes each relay message
✅ Load Balancing: Automatic distribution between available target clusters
✅ No Duplicates: Each message published once to single subject
✅ Zero Misconfiguration: Explicit configuration prevents routing errors
✅ Flexible: Easy to add/remove clusters or customize routing rules