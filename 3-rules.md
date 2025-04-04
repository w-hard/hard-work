1.1 Избавляться от точек генерации исключений, запрещая соответствующее ошибочное поведение на уровне интерфейса класса.

Сторонняя система присылает сообщения, в которых есть поле label.
Для каждого label есть определенное правило парсинга. Нужно распасить label и полученное значение сохранить в поле type объекта Quotation.
Список возможных label закреплен на уровне enum LabelMasks. Но парсинг поля label происходит прямо в месте создания Quotation, и легко можно ошибиться и распарсить label так, что будет ошибка.
```
public enum LabelMasks {
    XMX("XMX."),
    PBK("PBK."),
    LMTC("LMTC.Q.");
    
    public final String name;
    
    public static Optional<LabelMasks> fromLabel(String label) {
        return Arrays.stream(values())
        .filter(m -> label.contains(m.name))
        .findFirst();
    }
}
//client

LabelMask mask = LabelMask.fromLabel(label).orElseThrow();

swith(mask) {
    case XMX, PBK -> { quotation.setType(message.getLabel().split("\\.")[1]); };
    case LMTC -> { quotation.setType(message.getLabel().split("\\.")[2]); };
}
```
В новом варианте логика парсинга сохраняется для каждого лейбла в виде функции в классе LabelMasks.

Теперь не нужен switch на стороне клиента:

```
public enum LabelMasks {
    XMX("XMX.", s -> s.split("\\.")[1] ),
    PBK("PBK.", s-> s.split("\\.")[1] ),
    LMTC("LMTC.Q.", s-> s.split("\\.")[2] );
    
    String name;
    UnaryOperator<String> typeExtractor;
    
    public static Optional<LabelMasks> fromLabel(String label) {
        return Arrays.stream(values())
        .filter(m -> label.contains(m.name))
        .findFirst();
    }
    public static Optional<String> getTypeFromLabel(String label) {
        return fromLabel(label).map(m -> m.typeExtractor.apply(label));
    }

}
//client
String type = LabelMasks.getTypeFromLabel(label).orElseThrow();
quotation.setType(type);
```
1.2
Есть таблица asset_param. Она хранит список дополнительных параметров для сущности Asset.
```
create table asset_param (
    asset_id varchar,
    param varchar,
    value varchar
);
```
Был репозиторий, использующий под капотом реализацию Spring Data Jpa, который возвращал сущность AssetParamEntity.
```
public interface AssetParamJpaRepository extends JpaRepository<AssetParamEntity, AssetParamEntityId> {
    Optional<AssetParamEntity> findByAssetIdAndParam(String assetId, String param);
}
//client
AssetParamEntity assetParam = findByAssetIdAndParam(assetId, "purchase_date").orElseThrow();
String value = assetParam.getValue();
//конвертируется в строку

```
В новом варианте в репозитории создан метод, который извлекает именно значение параметра purchase_date, поэтому вероятность ошибки минимальна.
А также создан сервис, где происходит конвертация строки в дату.
```
public interface AssetParamJpaRepository extends JpaRepository<AssetParamEntity, AssetParamEntityId> {
    @Query("SELECT e.value FROM AssetParamEntity e " +
    "WHERE e.assetId = :assetId and e.param = 'purchase_date'")
    Optional<String> getPurchaseDateForAsset(@Param("assetId") String assetId);
}

public class AssetParamService {

    private final AssetParamJpaRepository assetParamJpaRepository;

    public OffsetDateTime getPurchasegDateForAsset(String assetId) {
        Optional<String> purchaseDateOpt = assetParamJpaRepository.getPurchaseDateForAsset(assetId);
        if (purchaseDateOpt.isEmpty()) {
            return null;
        }
        try {
            LocalDateTime purchaseDate = LocalDateTime.parse(purchaseDateOpt.get(), DateTimeFormatter.ISO_LOCAL_DATE_TIME);
            return purchaseDate.atOffset(ZoneOffset.UTC);
        } catch (DateTimeParseException e) {
            throw new IllegalArgumentException("Cant parse purchase date");
        }
    }
}
```

2.3 Отказаться от дефолтных конструкторов без параметров, и передавать конструктору обязательные аргументы;

Есть класс AssetDealMessage. Это DTO, у него есть обязательные поля, без которых процессинг невозможен. Но у класса есть делотный конструктор.
В новом варианте все обязательные поля прописываются в конструкторе.

3.1 Избегать увлечнения примитивными типами данных.

При работе с FIX протоколом в теле сообщения использовались кастомные теги.
Например, тенор (период времени, оставшийся до даты погашения финансового инструмента), под тегом 7999.
```
message.getGroups(NoMDEntries.FIELD).forEach(group -> {
    Asset asset = Asset.builder()
        .tenorValue(getStrField(message, 7999).orElse(""))
        ...
        .build();
        assets.add(asset);
    });
```
Можно все кастомные теги переместить в enum.
```
public enum FixTag {

    TENOR(7999),
    PRICE(6777),;

    final int tag;
    
    FixTag(int tag) {
        this.tag = tag;
    }

    public int getTag() {
        return tag;
    }
}
//client
message.getGroups(NoMDEntries.FIELD).forEach(group -> {
    Asset asset = Asset.builder()
        .tenorValue(getStrField(message, FixTag.TENOR.getTag()).orElse(""))
        ...
        .build();
    assets.add(asset);
});

```
