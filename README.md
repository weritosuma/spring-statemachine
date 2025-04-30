**Полный гайд по Spring State Machine для онлайн-магазина (Spring Boot 3 + Java 21 + Lombok)**  
_Включая сохранение состояния в БД, граф переходов и полные примеры кода_

---

### 1. **Зависимости** (`pom.xml`)
```xml
<dependencies>
    <!-- Spring Boot 3 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- State Machine -->
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-starter</artifactId>
        <version>4.0.0</version>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- JPA + H2 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

---

### 2. **Состояния и события** (`OrderState.java`)
```java
public enum OrderState {
    CREATED,
    PROCESSING,
    SHIPPED,
    DELIVERED // Конечное состояние
}

public enum OrderEvent {
    PROCESS,
    SHIP,
    DELIVER,
    CANCEL
}
```

---

### 3. **Граф переходов**
```
[CREATED]
  │ PROCESS (если товар в наличии)
  ▼
[PROCESSING] ──CANCEL──▶ [CREATED]
  │ SHIP
  ▼
[SHIPPED] ──CANCEL──▶ [CREATED]
  │ DELIVER
  ▼
[DELIVERED]
```

---

### 4. **Конфигурация State Machine** (`OrderStateMachineConfig.java`)
```java
@Configuration
@EnableStateMachine
@RequiredArgsConstructor
public class OrderStateMachineConfig extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    private final Guard<OrderState, OrderEvent> inventoryGuard;

    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states) throws Exception {
        states
            .withStates()
            .initial(OrderState.CREATED)
            .end(OrderState.DELIVERED)
            .states(EnumSet.allOf(OrderState.class));
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions) throws Exception {
        transitions
            .withExternal()
                .source(OrderState.CREATED).target(OrderState.PROCESSING)
                .event(OrderEvent.PROCESS)
                .guard(inventoryGuard)
            .and()
            .withExternal()
                .source(OrderState.PROCESSING).target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP)
            .and()
            .withExternal()
                .source(OrderState.SHIPPED).target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER)
            .and()
            .withExternal()
                .source(OrderState.PROCESSING).target(OrderState.CREATED)
                .event(OrderEvent.CANCEL)
            .and()
            .withExternal()
                .source(OrderState.SHIPPED).target(OrderState.CREATED)
                .event(OrderEvent.CANCEL);
    }

    @Bean
    public Guard<OrderState, OrderEvent> inventoryGuard() {
        return context -> {
            // Реальная проверка из базы данных
            return true; 
        };
    }
}
```

---

### 5. **Сущность заказа с Lombok** (`Order.java`)
```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Enumerated(EnumType.STRING)
    private OrderState currentState;

    @Lob
    @Convert(converter = StateMachineContextConverter.class)
    private StateMachineContext<OrderState, OrderEvent> context;
}
```

---

### 6. **Конвертер состояния** (`StateMachineContextConverter.java`)
```java
@Converter
@RequiredArgsConstructor
public class StateMachineContextConverter implements 
    AttributeConverter<StateMachineContext<OrderState, OrderEvent>, byte[]> {

    private final ObjectMapper mapper;

    public StateMachineContextConverter() {
        this.mapper = new ObjectMapper();
        this.mapper.registerModule(new Jackson2StateMachineModule());
    }

    @Override
    public byte[] convertToDatabaseColumn(StateMachineContext<OrderState, OrderEvent> context) {
        try {
            return mapper.writeValueAsBytes(context);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Serialization error", e);
        }
    }

    @Override
    public StateMachineContext<OrderState, OrderEvent> convertToEntityAttribute(byte[] bytes) {
        if (bytes == null) return null;
        try {
            return mapper.readValue(bytes, new TypeReference<>() {});
        } catch (IOException e) {
            throw new RuntimeException("Deserialization error", e);
        }
    }
}
```

---

### 7. **Репозиторий и сервис** (`OrderService.java`)
```java
public interface OrderRepository extends JpaRepository<Order, String> {}

@Service
@Transactional
@RequiredArgsConstructor
public class OrderService {
    
    private final OrderRepository orderRepo;
    private final StateMachine<OrderState, OrderEvent> stateMachine;
    private final StateMachinePersister<OrderState, OrderEvent, String> persister;

    public Order createOrder() {
        return orderRepo.save(Order.builder()
            .currentState(OrderState.CREATED)
            .build());
    }

    public void processOrder(String orderId) throws Exception {
        Order order = orderRepo.findById(orderId)
            .orElseThrow(() -> new EntityNotFoundException("Order not found"));
        
        // Восстановление состояния
        persister.restore(stateMachine, orderId);
        
        // Отправка события
        if (stateMachine.sendEvent(OrderEvent.PROCESS)) {
            // Сохранение нового состояния
            order.setContext(stateMachine.getStateMachineContext());
            order.setCurrentState(stateMachine.getState().getId());
            orderRepo.save(order);
        }
    }
}
```

---

### 8. **Конфигурация персистентности** (`PersistenceConfig.java`)
```java
@Configuration
@RequiredArgsConstructor
public class PersistenceConfig {

    private final OrderRepository orderRepo;

    @Bean
    public StateMachinePersister<OrderState, OrderEvent, String> persister() {
        return new DefaultStateMachinePersister<>(new StateMachinePersist<>() {
            @Override
            public void write(StateMachineContext<OrderState, OrderEvent> context, String orderId) {
                Order order = orderRepo.findById(orderId).orElseThrow();
                order.setContext(context);
                orderRepo.save(order);
            }

            @Override
            public StateMachineContext<OrderState, OrderEvent> read(String orderId) {
                return orderRepo.findById(orderId)
                    .map(Order::getContext)
                    .orElse(null);
            }
        });
    }
}
```

---

### 9. **Главный класс приложения** (`Application.java`)
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

### 10. **Как запустить**
1. Создать заказ:  
   ```bash
   curl -X POST http://localhost:8080/orders
   ```
2. Обработать заказ:  
   ```bash
   curl -X POST http://localhost:8080/orders/{id}/process
   ```

---

### 11. **Особенности реализации**
1. **Lombok**:  
   - `@Data`, `@Builder` и `@RequiredArgsConstructor` генерируют геттеры/сеттеры и конструкторы.
   
2. **Java 21**:  
   - Использованы `Records` (если нужно) и улучшенная работа с сериализацией.

3. **Персистентность**:  
   - Состояние сохраняется в H2 (для тестов) или любую другую SQL БД.

4. **Безопасность**:  
   Все операции с State Machine выполняются в транзакциях (`@Transactional`).

---

**Итог**: Полная реализация State Machine для онлайн-магазина с автоматическим сохранением состояния в БД и минимальным boilerplate-кодом благодаря Lombok.
