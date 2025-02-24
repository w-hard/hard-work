1.1. Методы, которые используются только в тестах.

На этот пункт нашла лишь 1 пример, видимо на текущем проекте не распространена практика создавать такие методы.
Обнаруженный метод getMessages по сути является оберткой для вызова метода репозитория, поэтому для того, чтобы избавиться, достаточно было перенести зависимость на репозиторий и сам метод в вызывающий тест.
```
@Service
public class MessageService {
    private final MessageJpaRepository messageRepository;

    public List<MessageEntity> getMessages() {
        return messageRepository.findAll();
    }
}
```

1.3. У метода слишком большой список параметров.

Обнаружила написанный мной метод с 5-ю параметрами getIncome(...)
Это метод используется для расчета купона по облигации, вызывающий класс передавал в метод все, что необходимо для подсчета, 
а именно номинал облигации, процент доходности spread, параметр c, который определяет, учитывается ли 1-й день расчетного периода. 
Эти же параметры передавались в private метод calculateCoupon(...) для дальнейших расчетов, там было уже целых 9 параметров.
Но т.к. АТД Калькулятор создавался для каждой конкретной облигации, эти параметры можно было сразу при создании экземпляра Калькулятора сохранить как его поля, и не передавать каждый раз при расчете.
Добавила поля daysBefore, c, spread, nominal в класс CouponCalculator, и количество параметров getIncome стало 2, а calculateCoupon - 5.

Исходно:
```
public class IncomeCalculator {
    
    TreeMap<LocalDate, Rate> ratesTree;
    
    private BigDecimal getIncome(LocalDate calculatorStart, LocalDate calculationDate, int daysBefore, BigDecimal spread, BigDecimal nominal, int c) {
        BigDecimal sum = BigDecimal.ZERO;
        
        TreeMap<LocalDate, Rate> tempTree = new TreeMap<>(ratesTree);
        Map.Entry<LocalDate, Rate> next;
        
        LocalDate periodStart = calculatorStart;
        BigDecimal val = tempTree.pollFirstEntry().getValue().getValue();
        
        while (!tempTree.isEmpty()) {
            next = tempTree.pollFirstEntry();
            LocalDate periodEnd = next.getKey();
            sum = sum.add(calculateCoupon(val, periodStart, periodEnd, calculatorStart, calculationDate, daysBefore, spread, nominal, c));
            periodStart = next.getKey();
            val = next.getValue().getValue();
        }
        return sum;
    }

}
```
После:
```
public class IncomeCalculator {
    
    TreeMap<LocalDate, kRate> ratesTree;
    private int daysBefore;
    private int c;
    private BigDecimal spread;
    private BigDecimal nominal;
    
    
    public BigDecimal getIncome(LocalDate calculatorStart, LocalDate calculationDate) {
        ...
        while (!tempTree.isEmpty()) {
            ...
            sum = sum.add(calculateCoupon(val, periodStart, periodEnd, calculatorStart, calculationDate));
            ...
        }
        return sum;
    }

}
```
1.2. Цепочки методов. Метод вызывает другой метод, который вызывает другой метод, который вызывает другой метод, который вызывает другой метод... и далее и далее.
От инициализации продукта до расчета накопленного купонного дохода длинная цепочка методов, вызывающих друг друга.
Получилось уменьшить длину цепочки тем, что метод getIncome и остальные методы для расчета купонного дохода вынесла в АТД IncomeCalculator, который создается отдельно для каждого продукта.

```
class ProductMessageService {
    
    public List<Message> processMessage(Message message) {
        ...
        messagesToSend.addAll(handler.handle(message));
        }
    }
}
    
public class RequestEventMessageHandler implements MessageHandler<RequestMessage> {
    private final DelegatingMessageService delegatingMessageService;
    
    @Override
    public List<Event> handle(RequestMessage message) {
    ...
        return delegatingMessageService.produce(message.getBody());
    }
}

public class DelegatingMessageService {
    
    public List<ProductMessage> produce(ProductEvent<?, ?> productEvent) {
        ...
        return producers.stream()
            .flatMap(productMessageProducer -> productMessageProducer.produce(productEvent).stream())
            .toList();
    }
}

    public class IncomeProducer extends AbstractMessageProducer{
    
    @Override
    public List<IncomeMessage> produce(ProductEvent<Product, EventType> productEvent) {
    
        return productService.getActiveBondProducts()
            .stream()
            .flatMap(product -> createIncomMessage(product, accountingDate).stream())
            .toList();
    }
    
    public Optional<IncomeMessage> createIncomMessage(CompositeProductId compositeProductId, LocalDate accountingDate) {
        ...
        Coupon coupon = couponService.fetchCoupon(...);
        return buildIncomeEvent(product, accountingDate, coupon););
    }
    
    public Optional<IncomeMessage> buildIncomeEvent(Product product,LocalDate accountingDate, Coupon coupon) {
        ...
        Map<LocalDate, BigDecimal> predict = getIncome(calculatorStart, calculationDate, daysBefore, spread, nominal, c);
        // getIncome в свою очередь вызывает еще несколько методов для получения данных для подсчета, самого подсчета и округления полученного результата
    }
```
