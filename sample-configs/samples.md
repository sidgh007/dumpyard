
```` java

@Service
@Slf4j
public class RelayPublisherService {

    @Autowired
    private NatsPublisherFactory natsPublisherFactory;
    
    @Autowired
    private NatsConfig natsConfig;
    
    @Value("${cluster.id}")
    private String clusterId;
    
    private NatsPublisher relayPublisher;
    
    @PostConstruct
    public void initializePublisher() {
        NatsConfig.PublisherConfig.RelayPublisher config = natsConfig.getPublishers().getRelayPublisher();
        if (config.isEnabled()) {
            this.relayPublisher = natsPublisherFactory.createNatsPublisher(config.getSubject());
            log.info("Initialized relay publisher for cluster: {} on subject: {}", 
                    clusterId, config.getSubject());
        }
    }
    
    public void publishRelayFromInactiveStream(String streamId, String clientId, Message originalEgressMessage) {
        if (relayPublisher != null) {
            try {
                // Add relay headers to the original message
                Message relayMessage = Message.builder()
                        .data(originalEgressMessage.getData())
                        .headers(createRelayHeaders(streamId, clientId, originalEgressMessage))
                        .build();
                
                relayPublisher.publish(relayMessage);
                log.info("Published relay message - Stream: {}, Client: {}, Original Subject: {}", 
                        streamId, clientId, originalEgressMessage.getSubject());
            } catch (Exception e) {
                log.error("Error publishing relay message for stream: {}", streamId, e);
            }
        }
    }
    
    private Headers createRelayHeaders(String streamId, String clientId, Message originalMessage) {
        Headers headers = new Headers();
        headers.add("relay-stream-id", streamId);
        headers.add("relay-client-id", clientId);
        headers.add("relay-original-cluster", clusterId);
        headers.add("relay-timestamp", String.valueOf(System.currentTimeMillis()));
        headers.add("relay-original-subject", originalMessage.getSubject());
        
        // Copy original headers if they exist
        if (originalMessage.getHeaders() != null) {
            for (String key : originalMessage.getHeaders().keySet()) {
                headers.add("original-" + key, originalMessage.getHeaders().getFirst(key));
            }
        }
        
        return headers;
    }
}


----------

@Slf4j
public class RelayEventsConsumer implements Consumer<Message> {
    
    private final String sourceCluster;
    private final StreamDeliveryService streamDeliveryService;
    
    @Value("${cluster.id}")
    private String clusterId;
    
    public RelayEventsConsumer(String sourceCluster, StreamDeliveryService streamDeliveryService) {
        this.sourceCluster = sourceCluster;
        this.streamDeliveryService = streamDeliveryService;
    }
    
    @Override
    public void accept(Message natsMessage) {
        try {
            // Extract relay information from headers
            String streamId = getHeaderValue(natsMessage, "relay-stream-id");
            String clientId = getHeaderValue(natsMessage, "relay-client-id");
            String originalCluster = getHeaderValue(natsMessage, "relay-original-cluster");
            String originalSubject = getHeaderValue(natsMessage, "relay-original-subject");
            
            log.info("Processing relay from cluster: {} - Stream: {}, Client: {}, Original Subject: {}", 
                    sourceCluster, streamId, clientId, originalSubject);
            
            // Validate message
            if (!isValidRelayMessage(natsMessage, streamId, clientId, originalCluster)) {
                log.warn("Invalid relay message received from cluster: {}, skipping", sourceCluster);
                return;
            }
            
            // Process the relayed message directly
            processRelayedEvent(natsMessage, streamId, clientId);
            
            log.info("Successfully processed relay from cluster: {} for stream: {}", 
                    sourceCluster, streamId);
                    
        } catch (Exception e) {
            log.error("Error processing relay message from cluster: {} - Subject: {}", 
                    sourceCluster, natsMessage.getSubject(), e);
        }
    }
    
    private String getHeaderValue(Message message, String headerKey) {
        if (message.getHeaders() != null) {
            return message.getHeaders().getFirst(headerKey);
        }
        return null;
    }
    
    private boolean isValidRelayMessage(Message natsMessage, String streamId, String clientId, String originalCluster) {
        if (streamId == null || streamId.isEmpty()) return false;
        if (clientId == null || clientId.isEmpty()) return false;
        if (originalCluster == null || originalCluster.isEmpty()) return false;
        if (natsMessage.getData() == null) return false;
        
        // Don't process messages from own cluster
        if (clusterId.equals(originalCluster)) {
            log.warn("Received relay message from own cluster: {}, ignoring", clusterId);
            return false;
        }
        
        return true;
    }
    
    private void processRelayedEvent(Message natsMessage, String streamId, String clientId) {
        try {
            // Deliver the original message directly to active stream
            boolean delivered = streamDeliveryService.deliverToActiveStream(clientId, natsMessage);
            
            if (delivered) {
                log.info("Relay message delivered to active stream - Client: {}, Stream: {}", 
                        clientId, streamId);
            } else {
                log.warn("No active stream found for client: {} on cluster: {}, relay message dropped", 
                        clientId, clusterId);
            }
            
        } catch (Exception e) {
            log.error("Error processing relayed event for client: {}", clientId, e);
            throw e;
        }
    }
    
    @Override
    public void onStart() {
        log.info("Started relay consumer for source cluster: {} on target cluster: {}", 
                sourceCluster, clusterId);
    }
    
    @Override
    public void onStop() {
        log.info("Stopped relay consumer for source cluster: {} on target cluster: {}", 
                sourceCluster, clusterId);
    }
}


------


@Service
@Slf4j
public class StreamDeliveryService {
    
    @Autowired
    private ActiveStreamManager activeStreamManager;
    
    @Value("${cluster.id}")
    private String clusterId;
    
    /**
     * Attempts to deliver relayed message directly to an active stream
     */
    public boolean deliverToActiveStream(String clientId, Message relayedMessage) {
        try {
            // Check if we have an active stream for this client
            Optional<ActiveStream> activeStream = activeStreamManager.findActiveStreamByClient(clientId);
            
            if (activeStream.isPresent()) {
                // Deliver the original message directly to the stream
                activeStream.get().sendMessage(relayedMessage);
                
                log.debug("Delivered relay message to active stream - Client: {}, Cluster: {}", 
                         clientId, clusterId);
                return true;
            } else {
                log.debug("No active stream found for client: {} on cluster: {}", clientId, clusterId);
                return false;
            }
            
        } catch (Exception e) {
            log.error("Error delivering message to active stream - Client: {}, Cluster: {}", 
                     clientId, clusterId, e);
            return false;
        }
    }
}


----------------

@ConfigurationProperties(prefix = "nats")
@Data
@Component
@EnableConfigurationProperties
public class NatsConfig {
    
    @Autowired
    private Environment environment;
    
    @Value("${cluster.id}")
    private String clusterId;
    
    @Value("#{'${cluster.secondary-clusters}'.split(',')}")
    private List<String> secondaryClusters;
    
    private ConsumerConfig consumers = new ConsumerConfig();
    private PublisherConfig publishers = new PublisherConfig();
    
    // Get relay consumer configurations for secondary clusters
    public List<RelayConsumerConfig> getRelayConsumerConfigs() {
        List<RelayConsumerConfig> configs = new ArrayList<>();
        
        for (String cluster : secondaryClusters) {
            if (!cluster.trim().isEmpty()) {
                RelayConsumerConfig config = buildRelayConsumerConfig(cluster.trim());
                if (config != null) {
                    configs.add(config);
                }
            }
        }
        
        return configs;
    }
    
    private RelayConsumerConfig buildRelayConsumerConfig(String sourceCluster) {
        try {
            String subject = environment.getProperty("nats.consumers.relay-events." + sourceCluster + ".subject");
            String queueGroup = environment.getProperty("nats.consumers.relay-events." + sourceCluster + ".queue-group");
            
            if (subject != null && queueGroup != null) {
                return RelayConsumerConfig.builder()
                        .sourceClusterId(sourceCluster)
                        .subject(subject)
                        .queueGroup(queueGroup)
                        .enabled(consumers.getRelayEvents().isEnabled())
                        .build();
            } else {
                log.warn("Missing configuration for relay consumer from cluster: {}", sourceCluster);
                return null;
            }
        } catch (Exception e) {
            log.error("Error building relay consumer config for cluster: {}", sourceCluster, e);
            return null;
        }
    }
    
    @Data
    public static class ConsumerConfig {
        private RelayEvents relayEvents = new RelayEvents();
        
        @Data 
        public static class RelayEvents {
            private boolean enabled = true;
        }
    }
    
    @Data
    public static class PublisherConfig {
        private RelayPublisher relayPublisher = new RelayPublisher();
        
        @Data
        public static class RelayPublisher {
            private String subject;
            private boolean enabled = true;
        }
    }
    
    @Data
    @Builder
    public static class RelayConsumerConfig {
        private String sourceClusterId;
        private String subject;
        private String queueGroup;
        private boolean enabled;
    }
}


--------------

@Service
@Slf4j
public class ConsumerRegistrationService {
    
    @Autowired
    private NatsConsumerFactory natsConsumerFactory;
    
    @Autowired
    private NatsConfig natsConfig;
    
    @Autowired
    private StreamDeliveryService streamDeliveryService;
    
    @Value("${cluster.id}")
    private String clusterId;
    
    private List<NatsConsumer> registeredConsumers = new ArrayList<>();
    
    @PostConstruct
    public void registerAllConsumers() {
        log.info("Starting consumer registration for cluster: {}", clusterId);
        registerRelayConsumers();
        log.info("Consumer registration completed for cluster: {}", clusterId);
    }
    
    private void registerRelayConsumers() {
        List<NatsConfig.RelayConsumerConfig> configs = natsConfig.getRelayConsumerConfigs();
        
        if (configs.isEmpty()) {
            log.warn("No relay consumer configurations found for cluster: {}", clusterId);
            return;
        }
        
        log.info("Registering {} relay consumers for cluster: {}", configs.size(), clusterId);
        
        for (NatsConfig.RelayConsumerConfig config : configs) {
            if (config.isEnabled()) {
                registerSingleRelayConsumer(config);
            } else {
                log.info("Relay consumer disabled for source cluster: {}", config.getSourceClusterId());
            }
        }
    }
    
    private void registerSingleRelayConsumer(NatsConfig.RelayConsumerConfig config) {
        try {
            // Create the relay consumer
            RelayEventsConsumer consumer = new RelayEventsConsumer(
                    config.getSourceClusterId(), 
                    streamDeliveryService
            );
            
            // Create and start NATS consumer
            NatsConsumer natsConsumer = natsConsumerFactory.createNatsConsumer(
                    config.getSubject(),    // relay.from-c2
                    config.getQueueGroup(), // qgroup-relay-from-c2
                    consumer
            );
            
            natsConsumer.start();
            
            // Keep track of registered consumers for cleanup
            registeredConsumers.add(natsConsumer);
            
            log.info("✅ Registered relay consumer - Source: {}, Subject: {}, Queue: {}", 
                    config.getSourceClusterId(), config.getSubject(), config.getQueueGroup());
                    
        } catch (Exception e) {
            log.error("❌ Failed to register relay consumer for source cluster: {}", 
                    config.getSourceClusterId(), e);
            throw new RuntimeException("Consumer registration failed for cluster: " + config.getSourceClusterId(), e);
        }
    }
    
    @PreDestroy
    public void cleanup() {
        log.info("Cleaning up {} consumers for cluster: {}", registeredConsumers.size(), clusterId);
        
        for (NatsConsumer consumer : registeredConsumers) {
            try {
                consumer.stop();
                log.debug("Stopped consumer: {}", consumer);
            } catch (Exception e) {
                log.error("Error stopping consumer", e);
            }
        }
        
        registeredConsumers.clear();
        log.info("Consumer cleanup completed for cluster: {}", clusterId);
    }
}

------

# Primary cluster
cluster.id=c1
cluster.secondary-clusters=c2,c3

# Relay consumers (subscribe to other clusters)
nats.consumers.relay-events.enabled=true
nats.consumers.relay-events.c2.subject=relay.from-c2
nats.consumers.relay-events.c2.queue-group=qgroup-relay-from-c2
nats.consumers.relay-events.c3.subject=relay.from-c3
nats.consumers.relay-events.c3.queue-group=qgroup-relay-from-c3

# Relay publisher (publish own relays)
nats.publishers.relay-publisher.subject=relay.from-c1
nats.publishers.relay-publisher.enabled=true


----


@SpringBootApplication
@EnableConfigurationProperties(NatsConfig.class)
@ComponentScan(basePackages = {"com.yourpackage"})
public class EventGatewayApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(EventGatewayApplication.class, args);
    }
}

----