# Microservices Design Patterns - Order Service

## 4.1 Microservices Design Patterns

### Pattern 1: Saga Pattern (Distributed Transaction)

The Saga pattern orchestrates multi-step workflows across distributed services without relying on distributed transactions. It's ideal for long-running business processes like purchase order workflows.

```python
# Order Service uses Saga for Order approval workflow

class OrderCreationSaga:
    """
    Orchestrates multi-step Order workflow across services
    
    Steps:
    1. Validate Order with Order Service
    2. Reserve Inventory with Inventory Service
    3. Validate Budget with Finance Service
    4. Submit to Oracle with ERP Service
    """
    
    async def execute_order_creation_saga(self, order_request):
        saga_id = uuid.uuid4()
        saga_state = SagaState(saga_id)
        
        try:
            # Step 1: Order Validation
            step1 = await self.order_service.validate_order(order_request)
            saga_state.add_step("order_validation", step1.id)
            
            # Step 2: Inventory Reservation
            step2 = await self.inventory_service.reserve_items(
                order_request.line_items
            )
            saga_state.add_step("inventory_reservation", step2.reservation_id)
            
            # Step 3: Budget Validation
            step3 = await self.finance_service.validate_budget(
                order_request.total_amount,
                order_request.cost_center
            )
            saga_state.add_step("budget_validation", step3.approval_id)
            
            # Step 4: Oracle Submission
            step4 = await self.erp_service.submit_order_to_oracle(order_request)
            saga_state.add_step("oracle_submission", step4.oracle_order_id)
            
            # All steps succeeded
            saga_state.mark_completed()
            
            # Publish success event
            await self.kafka_producer.publish_event(
                "SCM_Order_Approved",
                {
                    "sagaId": saga_id,
                    "oracleOrderId": step4.oracle_order_id,
                    "status": "APPROVED"
                }
            )
            
            return saga_state
            
        except Exception as e:
            # Compensate in reverse order
            await self._compensate_saga(saga_state, e)
            raise OrderSagaException(saga_state, e)
    
    async def _compensate_saga(self, saga_state, error):
        """Undo steps in reverse order (compensation logic)"""
        for step in reversed(saga_state.completed_steps):
            try:
                if step.name == "oracle_submission":
                    await self.erp_service.cancel_order(step.reference_id)
                elif step.name == "inventory_reservation":
                    await self.inventory_service.release_reservation(
                        step.reference_id
                    )
                elif step.name == "budget_validation":
                    await self.finance_service.release_budget(
                        step.reference_id
                    )
            except Exception as comp_error:
                # Log compensation failure for manual intervention
                await self.alert_operations_team(saga_state, comp_error)
```

**Key Benefits:**
- Supports long-running processes
- Loose coupling between services
- Easy to understand business workflow
- Clear compensation (rollback) logic

**Challenges:**
- Complexity: Must handle partial failures
- Compensation must be idempotent
- Requires careful state management
- Difficult to debug distributed transactions

---

### Pattern 2: Circuit Breaker (Resilience)

The Circuit Breaker pattern prevents cascading failures by failing fast when downstream services are unavailable. It acts like an electrical circuit breaker - stops requests when the service is overloaded.

```java
@Component
public class OracleServiceCircuitBreaker {
    private final CircuitBreaker circuitBreaker;
    private final OracleSCMClient oracleSCMClient;
    
    public OracleServiceCircuitBreaker(OracleSCMClient client) {
        this.oracleSCMClient = client;
        
        // Configure circuit breaker
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)        // Open if 50% fail
            .waitDurationInOpenState(Duration.ofSeconds(60))  // Try again after 60s
            .permittedNumberOfCallsInHalfOpenState(3)  // Allow 3 calls in half-open
            .automaticTransitionFromOpenToHalfOpenEnabled(true)
            .build();
        
        this.circuitBreaker = CircuitBreaker.of("oracle-scm", config);
        
        // Add event listeners
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> {
                logger.warn("Circuit breaker state: {} -> {}",
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState());
                
                // Alert ops team if opened
                if (event.getStateTransition().getToState() == State.OPEN) {
                    alertOpsTeam("Oracle SCM service circuit breaker opened");
                }
            });
    }
    
    public OracleOrderResponse createOrder(OrderRequest request) {
        try {
            return circuitBreaker.executeSupplier(() ->
                oracleSCMClient.createOrder(request)
            );
        } catch (CallNotPermittedException e) {
            // Circuit is open, fail fast
            logger.error("Circuit breaker open, cannot call Oracle service");
            throw new ServiceUnavailableException("Oracle SCM service temporarily unavailable");
        }
    }
}
```

**Circuit Breaker States:**

| State | Behavior | Transition |
|-------|----------|-----------|
| **CLOSED** | Requests pass through normally | → OPEN if failure rate exceeds 50% |
| **OPEN** | All requests fail immediately | → HALF_OPEN after 60s wait |
| **HALF_OPEN** | Allow test requests (3 calls) | → CLOSED if successful, OPEN if fails |

**Benefits:**
- Prevents cascading failures
- Fails fast, reduces wasted resources
- Automatic recovery
- Alerts ops teams proactively

**Use Cases:**
- Call to external APIs (Oracle SCM)
- Database connections
- Third-party services

---

### Pattern 3: Event Sourcing (Audit Trail)

Event Sourcing stores every state change as an immutable event. Instead of storing current state, we store the history of what happened. This creates a complete audit trail and enables state reconstruction.

```java
@Component
public class OrderEventSourcingService {
    
    @Autowired
    private OrderEventRepository eventRepository;
    
    @Autowired
    private KafkaProducerService kafkaProducer;
    
    /**
     * Store every state change as an event
     * This creates a complete audit trail
     */
    public void recordOrderEvent(
        String orderId,
        OrderEvent event) {
        
        // Store event in event store (audit table)
        OrderEventRecord eventRecord = new OrderEventRecord();
        eventRecord.setOrderId(orderId);
        eventRecord.setEventType(event.getType());
        eventRecord.setEventData(event.getData());
        eventRecord.setTimestamp(LocalDateTime.now());
        eventRecord.setActor(getCurrentUser());
        
        eventRepository.save(eventRecord);
        
        // Publish to Kafka for downstream consumers
        kafkaProducer.publishEvent(
            "SCM_Order_Events",
            eventRecord
        );
    }
    
    /**
     * Rebuild current state from events
     * Useful for debugging and compliance audits
     */
    public OrderState rebuildStateFromEvents(String orderId) {
        List<OrderEventRecord> events =
            eventRepository.findByOrderIdOrderByTimestampAsc(orderId);
        
        OrderState state = new OrderState();
        
        for (OrderEventRecord event : events) {
            state.apply(event);
        }
        
        return state;
    }
    
    /**
     * Example events for Order lifecycle:
     * - ORDER_CREATED: Initial order created
     * - ORDER_VALIDATED: Passed all validations
     * - ORDER_SUBMITTED_TO_ORACLE: Sent to ERP
     * - ORDER_APPROVED_IN_ORACLE: Approved in ERP
     * - ORDER_RECEIPTED: Goods received
     * - ORDER_INVOICED: Invoice created
     * - ORDER_PAID: Payment processed
     * - ORDER_CANCELLED: Order cancelled with reason
     */
}
```

**Event Store Benefits:**
- Complete audit trail (compliance requirement)
- Time-travel debugging (replay to any point)
- Event-driven architecture enablement
- Temporal queries ("show all orders from Q1 2024")

**Trade-offs:**
- Requires event versioning strategy
- Database grows continuously
- Must implement event projection/views
- Complex to query current state

---

### Pattern 4: CQRS (Command Query Responsibility Segregation)

CQRS separates read and write operations into different models. Write side handles state changes (commands), read side handles queries with an optimized denormalized view.

```java
// ============================================================
// WRITE SIDE: Command (State Changes)
// ============================================================

@Component
public class OrderCommandService {
    
    @Autowired
    private KafkaProducerService kafkaProducer;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Transactional
    public void createOrder(CreateOrderCommand command) {
        // Write to local DB
        Order order = new Order();
        order.setOrderNumber(command.getOrderNumber());
        order.setSupplier(command.getSupplier());
        order.setTotalAmount(command.getTotalAmount());
        order.setStatus(OrderStatus.DRAFT);
        
        orderRepository.save(order);
        
        // Publish event to sync read model
        kafkaProducer.publishEvent(
            "SCM_Order_Commands",
            new OrderCreatedEvent(order)
        );
    }
    
    @Transactional
    public void approveOrder(String orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        
        order.setStatus(OrderStatus.APPROVED);
        orderRepository.save(order);
        
        kafkaProducer.publishEvent(
            "SCM_Order_Commands",
            new OrderApprovedEvent(order)
        );
    }
}

// ============================================================
// READ SIDE: Query (Optimized for fast reads)
// ============================================================

@Component
public class OrderQueryService {
    
    @Autowired
    private OrderReadRepository readRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    /**
     * Query read model - optimized for fast reads
     * Read model is denormalized for specific query patterns
     */
    public List<OrderReadModel> queryOrdersBySupplier(String supplierId) {
        
        // Try cache first (very fast)
        String cacheKey = "orders:supplier:" + supplierId;
        List<?> cached = (List<?>) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return (List<OrderReadModel>) cached;
        }
        
        // Query read model (optimized database view)
        List<OrderReadModel> results =
            readRepository.findBySupplierId(supplierId);
        
        // Cache for 5 minutes
        redisTemplate.opsForValue().set(cacheKey, results, Duration.ofMinutes(5));
        
        return results;
    }
    
    /**
     * Subscribe to events to keep read model in sync
     * This is an eventual consistency model
     */
    @KafkaListener(topics = "SCM_Order_Events",
                   groupId = "order-query-service-group")
    public void onOrderEvent(OrderEventRecord event) {
        // Update read model with new event
        readRepository.updateFromEvent(event);
        
        // Invalidate cache so next query gets fresh data
        String cacheKey = "orders:supplier:" + event.getSupplierId();
        redisTemplate.delete(cacheKey);
        
        logger.info("Updated read model for order: {}", event.getOrderId());
    }
}
```

**CQRS Architecture:**

```
Write Path:
User Command → OrderCommandService → Database + Kafka Event

Read Path:
User Query → OrderQueryService → Redis Cache → OrderReadRepository → Database
                                     ↑ (eventual sync)
                                   Kafka Event Stream
```

**Benefits:**
- Independent scaling: Read and write can scale separately
- Query optimization: Read model tailored for specific queries
- Caching friendly: Easy to cache denormalized read model
- Event-driven: Natural fit with event sourcing

**Trade-offs:**
- Eventual consistency: Reads lag behind writes
- Complexity: Two separate models to maintain
- Requires event synchronization logic
- Cache invalidation challenges

---

## 4.2 Pattern Comparison & Use Cases

| Pattern | Use Case | Consistency | Complexity | Scalability |
|---------|----------|-------------|-----------|------------|
| **Saga** | Long-running workflows, distributed transactions | Eventual | High | High |
| **Circuit Breaker** | Resilience, prevent cascading failures | Strong | Low | Medium |
| **Event Sourcing** | Audit trail, compliance, state reconstruction | Strong | High | High |
| **CQRS** | Complex queries, read-heavy workloads | Eventual | High | Very High |

---

## 4.3 Recommended Pattern Combination for Order Service

### Hybrid Approach:

1. **Saga Pattern** for order creation workflow
   - Orchestrates: Validation → Inventory → Finance → Oracle
   - Handles compensation if any step fails

2. **Circuit Breaker** for Oracle SCM resilience
   - Protects against Oracle API failures
   - Fails fast with user-friendly errors

3. **Event Sourcing** for audit trail
   - Records all order state changes
   - Enables compliance audits and debugging

4. **CQRS** for order queries
   - Fast searches (by supplier, date, status)
   - Caching-friendly denormalized views

### Benefits of Combined Approach:

✅ **Reliability**: Saga + Circuit Breaker handle failures  
✅ **Auditability**: Event Sourcing tracks all changes  
✅ **Performance**: CQRS enables caching and optimized reads  
✅ **Scalability**: Each component scales independently  
✅ **Maintainability**: Clear separation of concerns  

---

## 4.4 Implementation Roadmap

### Phase 1: Foundation (Week 1-2)
- Implement Circuit Breaker for Oracle calls
- Add basic event publishing to Kafka
- Set up event store (audit table)

### Phase 2: Workflows (Week 3-4)
- Implement Saga orchestrator for order creation
- Add compensation handlers
- Build workflow state machine

### Phase 3: Optimization (Week 5-6)
- Implement CQRS read model
- Add Redis caching layer
- Create optimized query views

### Phase 4: Advanced (Week 7+)
- Full Event Sourcing implementation
- Event replay and time-travel debugging
- CQRS event synchronization
- Monitoring and alerting

---

## References
- [Microservices Patterns - Chris Richardson](https://microservices.io/patterns/index.html)
- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
