Есть метод, который получает на вход объект Quote и строку label, ищет в базе данных соответствующую сущность asset, делает проверки и возвращает id найденного asset. 
```
public Optional<String> getAssetId(String label, Quote quote) {
    log.info("Get assetId for {} {}", label, quote.getAssetType());
    List<AssetConfigEntity> assetConfigList = assetConfigRepository.findByLabelAndSource(label, ExternalSystem.BBK);
    QuoteType quoteType = quote.getQuoteType() == 0 ? QuoteType.BID : QuoteType.ASK;
    AssetType assetType = AssetType.valueOf(quote.getAssetType());

    assetConfigList = assetConfigList.stream()
            .filter(con -> c.getQuoteType() == quoteType)
            .filter(c -> c.getAssetType() == assetType)
            .toList();

    if (assetConfigList.size() == 1) {
        log.info("Asset id = {}", assetConfigList.get(0).getId());
        return Optional.of(assetConfigList.get(0).getId());
    } else if (assetConfigList.isEmpty()) {
        log.info("Can't resolve asset.");
        return Optional.empty();
    }
    throw new IllegalArgumentException("There is more that one asset found: " + assetConfigList);
}
```
Как можно вынести ограничения на более высокий уровень:

1. Можно установить contraint на уровне БД, который обеспечивает уникальное сочетание asset_type и quote_type в таблице assets.
   Так избавимся от выбрасывания исключения IllegalArgumentException.

2. В репозитории AssetConfigRepository можно сделать метод `Optional<AssetConfigEntity> findByLabelAndSourceAndQuoteTypeAndAssetType(String label, ExternalSystem system, QuoteType quoteType, AssetType assetType)`, который будет возвращать объект assetConfigOpt. В результате уйдет фильтрация assetConfigList.
3. А также уйдет проверка `if(assetConfigList.size() == 1) {...} else {...}`, т.к. можно будет записать в функциональном стиле: `return assetConfigOpt.map(AssetConfigEntity::getId)`;

```
public Optional<String> getAssetId(String label, Quote quote) {
    log.info("Get assetId for {} {}", label, quote.getAssetType());
    QuoteType quoteType = quote.getQuoteType() == 0 ? QuoteType.BID : QuoteType.ASK;
    AssetType assetType = AssetType.valueOf(quote.getAssetType());
    Optional<AssetConfigEntity> assetConfigOpt = assetConfigRepository.findByLabelAndSourceAndQuoteTypeAndAssetType(label, ExternalSystem.BBK, quoteType, assetType);
    return assetConfigOpt.map(AssetConfigEntity::getId);
}
```
3. В нашем коде 0 всегда кодирует цену покупки (bid), а 1 -- цену продажи (ask), поэтому можем вынести код цены в поле в enum QuoteType :
```
public enum QuoteType {
    BID(0),
    ASK(1);
    int code;
    Side(int code) { this.code = code; }
    public static QuoteType getByCode(int code) {
        return Arrays.stream(values()).filter(s -> s.code == code).findFirst().orElseThrow(() -> new IllegalArgumentException("Invalid code for QuoteType"));
    }
}
```
В итоге код превращается в такой:

```
public Optional<String> getAssetId(String label, Quote quote) {
    QuoteType quoteType = QuoteType.getByCode(quote.getQuoteType());
    AssetType assetType = AssetType.valueOf(quote.getAssetType());
    return assetConfigRepository.findByLabelAndSourceAndQuoteTypeAndAssetType(label, ExternalSystem.BBK, quoteType, assetType)
            .map(AssetConfigEntity::getId);;
}
```
4. Есть код, который валидирует входящее сообщение quickfix.Message (из библиотеки QuickFIX/J), которое представляет собой десериализованное сообщение протокола FIX. 
Проверяется, что сообщение содержит тег 55(Symbol), который содержит название тикера в протоколе FIX.
```

public List<MessageValidationError> validate(Message msg) {
    List<MessageValidationError> errors = new ArrayList<>();

    boolean isParsedCorrectly = msg.getException() == null;
    boolean isSymbolPresent = MessageUtil.extractField(msg, Symbol.FIELD).isPresent();

    if (!isParsedCorrectly) {
        errors.add("Message parsed with errors");
    }
    if(!isSymbolPresent) {
        errors.add("No symbol present.");
    }
    return errors;
}
```

Но как оказалось, проверку на непустой Symbol можно поместить в файл DataDictionary (QuickFIX-DataDictionary.xml), который будет автоматически валидировать входящее FIX сообщение.
Его можно прописать в конфигурации: DataDictionary=QuickFIX-DataDictionary.xml. 
Ограничение будет записано так:

`<field number="55" name="Symbol" type="STRING" required="yes"/>`

А вместо вызова метода validate(Message msg), можно просто получить ошибки из самого сообщения: msg.getException(), где в случае отсутствия Symbol будет соответствующее сообщение. 

