**Полный гайд по Spring State Machine для онлайн-магазина (Spring Boot 3 + Java 21 + Lombok)**  
_Включая сохранение состояния в БД, Guards и бизнес-логику_

---

### **1. Зависимости** (`pom.xml`)
```xml
<dependencies>
    <!-- Spring Boot 3 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-starter</artifactId>
        <version>4.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.statemachine</groupId>
        <artifactId>spring-statemachine-data-jpa</artifactId>
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

### **2. Состояния и события**
```java
public enum OrderState {
    CREATED, PROCESSING, SHIPPED, DELIVERED
}

public enum OrderEvent {
    PROCESS, SHIP, DELIVER, CANCEL
}
```

---

### **3. Граф переходов**
```
[CREATED] → PROCESS (если товары в наличии) → [PROCESSING]  
[PROCESSING] → SHIP (если оплачено) → [SHIPPED]  
[SHIPPED] → DELIVER (если адрес валиден) → [DELIVERED]  
[PROCESSING/SHIPPED] → CANCEL → [CREATED]
```

---

### **4. Сущность `Order` с Lombok**  
```java
@Entity
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Enumerated(EnumType.STRING)
    private OrderState state;

    @Lob
    @Convert(converter = StateMachineContextConverter.class)
    private StateMachineContext<OrderState, OrderEvent> context;

    private boolean paid;
    private String shippingAddress;

    @ElementCollection
    private List<OrderItem> items;
}

@Embeddable
@Data
@Builder
public class OrderItem {
    private String productId;
    private boolean inStock;
}
```

---

### **5. Конфигурация State Machine**
```java
@Configuration
@EnableStateMachine
@RequiredArgsConstructor
public class OrderStateMachineConfig extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    private final Guard<OrderState, OrderEvent> inventoryGuard;
    private final Guard<OrderState, OrderEvent> paymentGuard;
    private final Guard<OrderState, OrderEvent> addressGuard;

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
            .withExternal().source(OrderState.CREATED).target(OrderState.PROCESSING)
                .event(OrderEvent.PROCESS).guard(inventoryGuard)
            .and().withExternal().source(OrderState.PROCESSING).target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP).guard(paymentGuard)
            .and().withExternal().source(OrderState.SHIPPED).target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER).guard(addressGuard)
            .and().withExternal().source(OrderState.PROCESSING).target(OrderState.CREATED)
                .event(OrderEvent.CANCEL)
            .and().withExternal().source(OrderState.SHIPPED).target(OrderState.CREATED)
                .event(OrderEvent.CANCEL);
    }
}
```

---

### **6. Guards (Бизнес-логика)**  
```java
@Configuration
public class GuardsConfig {

    @Bean
    public Guard<OrderState, OrderEvent> inventoryGuard(OrderRepository repo) {
        return context -> {
            String orderId = (String) context.getMessageHeader("orderId");
            return repo.findById(orderId)
                .map(order -> order.getItems().stream().allMatch(OrderItem::isInStock))
                .orElse(false);
        };
    }

    @Bean
    public Guard<OrderState, OrderEvent> paymentGuard(OrderRepository repo) {
        return context -> {
            String orderId = (String) context.getMessageHeader("orderId");
            return repo.findById(orderId).map(Order::isPaid).orElse(false);
        };
    }

    @Bean
    public Guard<OrderState, OrderEvent> addressGuard() {
        return context -> {
            String address = (String) context.getExtendedState().get("shippingAddress");
            return address != null && !address.trim().isEmpty();
        };
    }
}
```

---

### **7. Сервис с транзакциями**  
```java
@Service
@Transactional
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    private final StateMachine<OrderState, OrderEvent> stateMachine;
    private final StateMachinePersister<OrderState, OrderEvent, String> persister;

    public Order createOrder(List<OrderItem> items) {
        return orderRepo.save(Order.builder()
            .state(OrderState.CREATED)
            .items(items)
            .build());
    }

    public void processOrder(String orderId) throws Exception {
        persister.restore(stateMachine, orderId);
        stateMachine.sendEvent(MessageBuilder.withPayload(OrderEvent.PROCESS)
            .setHeader("orderId", orderId)
            .build());
        persister.persist(stateMachine.getStateMachineContext(), orderId);
    }

    public void confirmPayment(String orderId) {
        orderRepo.findById(orderId).ifPresent(order -> {
            order.setPaid(true);
            orderRepo.save(order);
        });
    }
}
```

---

### **8. Персистентность**  
```java
@Converter
public class StateMachineContextConverter implements 
    AttributeConverter<StateMachineContext<OrderState, OrderEvent>, byte[]> {

    private static final ObjectMapper mapper = new ObjectMapper()
        .registerModule(new Jackson2StateMachineModule());

    @Override
    public byte[] convertToDatabaseColumn(StateMachineContext<OrderState, OrderEvent> context) {
        try {
            return mapper.writeValueAsBytes(context);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Ошибка сериализации", e);
        }
    }

    @Override
    public StateMachineContext<OrderState, OrderEvent> convertToEntityAttribute(byte[] bytes) {
        try {
            return bytes != null ? mapper.readValue(bytes, new TypeReference<>() {}) : null;
        } catch (IOException e) {
            throw new RuntimeException("Ошибка десериализации", e);
        }
    }
}
```

---

### **9. Как это работает**  
1. **Создание заказа**:  
   - `POST /orders` → сохраняет заказ в статусе `CREATED`.
2. **Обработка**:  
   - `POST /orders/{id}/process` → проверяет наличие товара (Guard), переводит в `PROCESSING`.
3. **Отправка**:  
   - `POST /orders/{id}/ship` → проверяет оплату (Guard), переводит в `SHIPPED`.
4. **Доставка**:  
   - `POST /orders/{id}/deliver` → проверяет адрес (Guard), переводит в `DELIVERED`.

---

**Итог**:  
- Guards реализуют ключевые бизнес-правила.  
- Состояние автоматически сохраняется в БД через JPA.  
- Lombok уменьшает объем шаблонного кода.  
- Полный цикл работы онлайн-магазина: от создания заказа до доставки.
