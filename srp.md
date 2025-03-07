1. Дано: строка длинная за счет того, что здесь создается объект с длинным списком параметров, а некоторые параметры преобразуются здесь же.
```
events.add(new GenericEvent(number, updated.txId(), ExtractionUtils.parseTimestamp(deal.timestamp()), null, null, EventType.DEAL, map(balance, deal, operationId, wallet))); 

public GenericEvent(long number, String txId, OffsetDateTime timestamp, Integer index, Integer indexInBlock, EventType eventType, T event) {...}
```
Возможно, здесь можно было бы применить паттерн builder.
```
GenericEvent event = GenericEvent.builder()
    .number(number) 
    .txId(updated.txId()) 
    .timestamp(ExtractionUtils.parseTimestamp(deal.timestamp())) 
    .eventType(EventType.DEAL) 
    .event(map(balance, deal, operationId, wallet)) 
    .build();
events.add(event); 

```
2. Дано: несколько вызовов функций делают строку не очень понятной.
```
Content content = context.getContent(processedEvent.event().askRequest().request().tokens()[0]);
 ```
Можно вынести вычисления в отдельную функцию.
```
Content content = getContent(processedEvent); 

private Content getContent(Event<Ask> processedEvent) { 
    AskEvent ask = processedEvent.event() ; 
    SignedRequest askRequest = ask.askRequest().request(); 
    return context.getContent(askRequest.tokens()[0]); 
}
```
3. Дано: в строке производятся преобразования, сначала деление, потом округление, потом конвертация BigDecimal to double.
```
balance.setQuote(BigDecimal.valueOf(balance.getSum()).divide(BigDecimal.valueOf(amount)).setScale(2, RoundingMode.HALF_UP).doubleValue());
```
Кажется, что если конструирование цены сделать при инициализации отдельной переменной, то это улучшит читаемость и будет соответствовать принципу SRP.
```
Double quoteBalanceValue = BigDecimal.valueOf(balance.getSum()) 
    .divide(BigDecimal.valueOf(amount)) 
    .setScale(2, RoundingMode.HALF_UP) 
    .doubleValue(); 
balance.setQuote(quoteBalanceValue);
```
4. Дано: в условии 2 проверки, но вторая не очень очевидная.
```
if (wallet.getRole() == TRADE || Arrays.stream(exchange.holders()).anyMatch(holder -> wallet.isActive(holder.owner()))) {...}
 ```
Тут возможно стоит вынести вторую проверку в отдельную переменную типа boolean, тогда условие будет легко понять.
```
boolean isActiveHolder = Arrays.stream(exchange.holders()) 
.anyMatch(holder -> wallet.isActive(holder.owner())); 

if (wallet.getRole() == TRADE || isActiveHolder) {
```
5. Дано: в строке комбинация 2-х отрицаний и логическое И. Чтобы понять, что имеется в виду, приходится задуматься, хотя строка сама по себе короткая.
```
return !end.isBefore(checkingDate) && !start.isAfter(checkingDate); // проверяем что дата внутри диапазона, включая начало и конец
```
Возможно стоит убрать отрицание, и дополнительно проверять что дата равна началу или концу диапазона, т.к. методы isAfter, isBefore используют строгое сравнение.
```
boolean isDateBetween = date.isAfter(start) && date.isBefore(end); 
return isDateBetween || date.equals(start) || date.isEqual(end);
   ```
6. Дано: очень длинное название метода в репозитории. Spring data автоматически создает sql-запрос исходя из параметров, которые используются в названии метода.
  Но сам метод становится нечитабельным.
```
Optional<AssetEntity> findFirstByUser_UserIdAndProductIdAndCreateDateLessThanEqualOrderByCreateDateDescVersionDesc(String userId, String productId, LocalDate createDate);
```
Имя метода делаем более коротким, а запрос пишем вручную, не используя генерацию из Spring Data.
```
@Query("SELECT e FROM AssetEntity e 
        where userId = :userId 
        and productId = :productId 
        and createDate <= :createDate 
        order by createDate desc 
        and version desc 
        limit 1") 
List<AssetEntity> findFirstAssetEarlierThanCreateDate(String userId, String productId, LocalDate createDate);
```
