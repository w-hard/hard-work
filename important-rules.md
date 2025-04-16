1. Код содержит 4 повторяющихся инструкции try-catch.
```
public void scheduleImport(Boolean blockOnly) {
    if (blockOnly) {
        log.info("Execute importerService.importBlock(EVENTS) ...");
        try {
            importerService.importBlock(EVENTS);
        } catch (Exception e) {
            log.error("Failed to execute importerService.importBlock(EVENTS)", e);
        }

        log.info("Execute importerService.importBlock(DEALS) ...");
        try {
            importerService.importBlock(DEALS);
        } catch (Exception e) {
            log.error("Failed to execute importerService.importBlock(DEALS)", e);
        }
    } else {
        log.info("Execute importerService.import(EVENTS) ...");
        try {
            importerService.import(EVENTS);
        } catch (Exception e) {
            log.error("Failed to execute importerService.import(EVENTS)", e);
        }

        log.info("Execute importerService.import(DEALS) ...");
        try {
            importerService.import(DEALS);
        } catch (Exception e) {
            log.error("Failed to execute importerService.import(DEALS)", e);
        }
    }
}
```
После рефакторинга перешли от управляющей структуры try-catch, повторенной 4 раза, к функции высшего порядка executeImportFunction, которая принимает функцию, выполняющуюся в теле try. А также параметр функции типа ImportType, который также используется при логировании. 
```
public void scheduleImport(Boolean blockOnly) {
    blockOnly ? importDealsAndEvents(importerService::importBlock) : importDealsAndEvents(importerService::import);
}

private void importDealsAndEvents(Function<ImportType, ?> importFunction) {
    executeImportFunction(importFunction, EVENTS);
    executeImportFunction(importFunction, DEALS);

}
private void executeImportFunction(Function<ImportType, ?> importFunction, ImportType type) {
    log.info(String.format("Execute %s(%s) ...", importFunction.toString(), type.name()));
    try {
        importFunction.apply(type);
    } catch (Exception e) {
        log.error(String.format("Failed to execute %s(%s)", importFunction, type.name()), e);
    }
}
```

2. В данном коде есть цикл, вложенный в if, и доступ к элементам коллекции по индексу.
```
public List<TLSystemPeriod> producePeriods(StaticData staticData, String assetId) {

    if (staticData instanceof ProductStaticData productStaticData) {
        if (BOND.equals(productStaticData.getAssetType())) {
            List<Period> periods = periodService.findPeriodsByAssetId(assetId)
                    .stream()
                    .sorted(Comparator.comparing(Period::getStart))
                    .toList();
            List<TLSystemPeriod> tlsPeriods = new ArrayList<>();

            for (int i = 1; i <= periods.size(); i++) {
                Period period = periods.get(i - 1);
                tlsPeriods.add(TlSystemPeriod.builder()
                        .number(i)
                        .startDate(period.getStart())
                        .endDate(period.getEnd())
                        .build());
            }
            return tlsPeriods;
        }
    }
    return List.of();
}
```

Добавляем в класс ProductStaticData статический метод, где инкапсулируем попытку приведения экземпляра предка StaticData к дочернему классу.
Получение списка periods записывается короче, т.к. сортировку можно осуществить при извлечении сущностей Period из базы с помощью Spring Data-реализации метода findPeriodsByAssetIdOrderByStart(...).
Создание объекта TLSystemPeriod можно вынести в отдельный метод getTlsPeriod(...), и вместо цикла использовать IntStream. Также избавились от цикла внутри if.

```
public List<TLSystemPeriod> produceTlsPeriods(StaticData staticData, String assetId) {

    ProductStaticData productStaticData = ProductStaticData.assignmentAttempt(staticData);
    if(productStaticData == null || !BOND.equals(productStaticData.getAssetType())) {
        return List.of();
    }
    List<Period> periods = periodService.findPeriodsByAssetIdOrderByStart(productId);
    return IntStream.rangeClosed(1, periods.size())
            .mapToObj(i -> getTlsPeriod(i, periods.get(i - 1)))
            .toList();

}
```

3. В данном коде есть switch, в каждой ветке которого выполняется код для извлечения сущности из различных репозиториев, либо выбрасывается ошибка.
Но с перевого взгляда сложно понять, что происходит, т.к. похоже на полотно.
```
private Event produceEvent(ProductEventEntity productEventEntity) {
    return switch (productEventEntity.getType()) {
        case IPO_ISSUE ->
                issueEventRepository.findById(new EventEntityId(EventEntityId.Type.IPO, productEventEntity.getEventId()))
                        .orElseThrow(() -> new NotFoundException("Failed to find IssueEventEntity by id=" + productEventEntity.getId()));
        case STOCK_ISSUE ->
                issueEventRepository.findById(new EventEntityId(EventEntityId.Type.STOCK, productEventEntity.getEventId()))
                        .orElseThrow(() -> new NotFoundException("Failed to find IssueEventEntity by id=" + productEventEntity.getEventId()));
        case INDEX -> indexEventJpaRepository.findById(productEventEntity.getEventId())
                .orElseThrow(() -> new NotFoundException("Failed to find IndexEventEntity by id=" + productEventEntity.getEventId()));
        case CONTRACT_UPDATED ->
                contractEventJpaRepository.findById(UUID.fromString(productEventEntity.getEventId()))
                        .orElseThrow(() -> new NotFoundException("Failed to find ContractEventEntity by id=" + productEventEntity.getEventId()));
        case PAYMENT -> paymentEventJpaRepository.findById(productEventEntity.getEventId())
                .orElseThrow(() -> new NotFoundException("Failed to find PaymentEventEntity by id=" + productEventEntity.getEventId()));
    };
}

```
Здесь постаралась применить факторизацию общих подвыражений - логику поиска сущности Event передавать как функцию в функцию высшего порядка getEvent, где также делать обработку ошибки.
Кажется, код стал более ясным, т.к. более унифицированный. В каждой ветке switch вызываем метод getEvent, и по его параметрам видно, в каком репозитории ищем и как получаем id для поиска.
А детали вроде обработки ошибки скрыты. 
```
private Event produceEvent(ProductEventEntity productEventEntity) {
    String eventId = productEventEntity.getEventId();
    EventType type = productEventEntity.getType();

    return switch (type) {
        case IPO_ISSUE ->
            getEvent(issueEventRepository::findById, id -> new EventEntityId(IPO, id), eventId, IssueEventEntity.class);
        case STOCK_ISSUE ->
            getEvent(issueEventRepository::findById, id -> new EventEntityId(STOCK, id), eventId, IssueEventEntity.class);
        case INDEX ->
            getEvent(indexEventJpaRepository::findById, Function.identity(), eventId, IndexEventEntity.class);
        case CONTRACT_UPDATED ->
            getEvent(contractEventJpaRepository::findById, UUID::fromString, eventId, ContractEventEntity.class);
        case PAYMENT ->
            getEvent(paymentEventJpaRepository::findById, Function.identity(), eventId, PaymentEventEntity.class);
    };
}
private <P> Event getEvent(Function<P, Optional<? extends Event>> function,
                                        Function<String, P> idGeneratorFunction,
                                        String eventId, Class<?> classP) {
    return function.apply(idGeneratorFunction.apply(eventId))
            .orElseThrow(() -> new NotFoundException(String.format("Failed to find %s by id = %s", classP.getSimpleName(), eventId)));
}
```
