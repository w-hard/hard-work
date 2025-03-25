#### 2.1. Класс слишком большой (нарушение SRP).

Есть класс ProductService > 250 строк.

У него 14 зависимостей на другие сервисы и репозитории. В классе 15 методов.

10 методов просто достают продукты из БД и применяют некий фильтр: 

getProducts, getDeletedProducts, getProduct, getInitializedProducts, getInitializedProductIds и т.д. .

5 методов меняют состояние объектов:
init, initProductSet, map, markProductAsDeleted, markProductSetAsDeleted.

Причем метод init самый большой, на 60 строк с цикломатической сложностью 10.

Возможно стоило бы вынести все модифицирующие методы в отдельный класс. Или хотя бы init и initProductSet методы в новый класс ProductInitializerService.


#### 2.2. Класс делает слишком мало.

Класс QuoteCalendarService, приведенный ниже, во-первых слишком маленький, во-вторых только вызывает методы других классов, практически ничего не добавляя. 
Можно просто в вызываюший клиент добавить QuoteService и CalendarClient в качестве зависимостей, и вызывать их методы напрямую.

```
public class QuoteCalendarService {

  private final QuoteService quoteService;
  private final CalendarService calendarService;

  public LocalDate getPreviousWorkingDay(LocalDate date){
    WorkingDayDate previousWorkingDayDate = calendarService.getPreviousWorkingDay(date);
    return previousWorkingDayDate.getDate();
  }
  
  public List<Quote> getQuoteForPeriod(String ticker, LocalDate start, LocalDate end) {
    return quoteService.getQuoteForPeriod(ticker, start, end);
  }
  
  public Quote getQuoteForDate(String code, LocalDate date) {
    return quoteService.getQuoteForDate(code, date);
  }
}
```

#### 2.3. В классе есть метод, который выглядит более подходящим для другого класса.

Есть класс BondIncomeCalculator, используется для расчета дохода по облигации. 
Содержит 3 метода, каждый подсчитывает доход и возвращает число: predictBondIncome, calculateBondIncomeForDate, calculateBondIncomeForPeriod.

Но класс также содержит метод isCouponAccumulationRecalculationNeeded, который определяет, нужно ли пересчитывать доход. 
Этот метод ничего не подсчитывает, он только сравнивает даты, поэтому его расположение в классе Калькулятор кажется не очень обоснованным. 
И вызывается он в другом контексте, нежели 3 первые метода, и лишь в 1 месте в коде. Кажется, что его можно просто перенести в клиент.

#### 2.4. Класс хранит данные, которые загоняются в него в множестве разных мест в программе.
Есть класс Deal. Изначально он создается на основе сообщения из Kafka из сторонней системы. Потом в него сохраняются данные, полученные из БД. Потом данные, полученные из кэша. В итоге сложно понять, что откуда пришло.

#### 2.5. Класс зависит от деталей реализации других классов.
При старте приложения, некоторые объекты Contract сохряняются в кэш.
В классах, где происходит извлечение данных из кэша, получаются длинные ветки If, где извлеченный из кэша contract приводятся к классам-наследникам. 
```
Contract contract = context.getCachedData().getContract(e.contractId());
if (contract instanceOf ... ) {...}
else if (contract instanceOf ... ) {...}
else if...
```

#### 2.6. Приведение типов вниз по иерархии.

В этом коде много приведения родительских типов к дочерним.
Например, ContractDto приводится к дочерним ShareContractDto, BondContractDto.
После этого у них извлекается список periods, которые трансформируются соответсвенно в SharePeriodEntity и BondPeriodEntity.
Код в итоге сложно воспринимается, а цикломатическая сложность метода = 10.
```
ProductEntity productEntity = ...
ContractDto contract = getContract();
ContractEntity contractEntity = contractMapper.map(contract);
contractRepository.save(contractEntity);

if (contract instanceof IssueContractDto issueContract) {
  for (var linkedContract : issueContract.getLinkedContracts()) {
  ContractEntity linkedContractEntity = contractMapper.map(linkedContract);
  contractRepository.save(linkedContractEntity);
  
  if (linkedContractEntity instanceof ShareContractEntity ShareContractEntity) {
  
    List<SharePeriodEntity> sharePeriodEntities = ((ShareContractDto) linkedContract).getPeriods().stream()
      .map(period -> (SharePeriodEntity) sharePeriodEntityMapper.map(period))
       ...
    
    } else if (linkedContractEntity instanceof BondContractEntity bondContractEntity) {

      List<BondPeriodEntity> bondPeriodEntities = ((BondContractDto) linkedContract).getPeriods().stream()
        .map(period -> (BondPeriodEntity) bondPeriodEntityMapper.map(period))
        ...

    }
    productRepository.save(productEntity);
    }
  }

```

#### 2.7. Когда создаётся класс-наследник для какого-то класса, приходится создавать классы-наследники и для некоторых других классов.

В примере выше был парсинг уже созданных классов-наследников ShareContractDto и BondContractDto.
При их создании приходится также создавать классы-наследники PeriodDto -- SharePeriodDto, BondPeriodDto.

#### 2.8. Дочерние классы не используют методы и атрибуты родительских классов, или переопределяют родительские методы.
Некоторые его потомки класса EventMessageHandler переиспользуют метод handle, но есть потомок, который переопределяет его как пустой.
```
public abstract class EventMessageHandler {
  @Override
  public void handle(T message) {
    // some logic
  }
}

public class TradingStartMessageHandler extends EventMessageHandler {
  @Override
  protected void handle(T message) {
    //do nothing
  }
}
```

#### 3.1. Одна модификация требует внесения изменений в несколько классов.

Есть класс BondContractDto, у него есть родительский класс ContractDto. Они находятся в библиотеке common-lib, которую используют другие микросервисы.
Допустим, понадобитлось добавить новое поле в класс BondContractDto. 
При этом нужно будет внести изменения еще в 3-х местах, в репозитории микросервиса contracts, где используются эти классы:
1. В классе BondContractEntity добавить новое поле.
2. В сервисе ContractService данные BondContractDto используются для создания другого объекта, нужно копировать туда новое поле.
3. В классе ContractMapper, который преобразует Entity в Dto для разных типов контрактов, надо дописать код, которые копирует значение нововго поля в BondContractDto из BondContractEntity.
   Здесь легко вообще упустить из внимания ContractMapper, потому что метод map принимает и возвращает родительские классы:
   ```
   ContractDto map(ContractEntity entity) {
     return switch (entity.getTemplateId()) {
       case Index -> ...
       case Bond -> ...
       case Share ->
     ...
   }
   ```
   И если забыть и не внести изменения в нужную ветку switch-a, на клиенте код не сломается, будет работать, но новое поле просто потеряется.

#### 3.2. Использование сложных паттернов проектирования.

В микросервисе происходит обработка сообщений из нескольких сторонних систем.
Есть общий для всех сообщений класс Processor, где находится логика парсинга и процессинга сообщения. 
Во время исполнения в программе столько экземпляров класса Processor, сколько типов сообщений.
Этот класс содержит несколько полей-интерфейсов (парсер, репозитории, валидатор и т.д.), куда инжектятся конкретые реализации - отдельные для каждого типа сообщений. 
Реализации для каждого типа сообщений лежат в отдельных модулях. 
И получается, в проекте 15 мавен-модулей, но некотрые модули содержат всего по 2-3 класса, что порой ухудшает понимание.
