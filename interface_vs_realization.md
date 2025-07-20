1. Примеры призрачного состояния.
   
В примере ниже призрачное состояние это локальные переменные isStartOfPeriodCorrect и isEndOfPeriodCorrect.
```
public class IncomeCalculator {

   LocalDate start;
   LocalDate end;
   
   /**
    Метод используется для проверки того, что данный калькулятор иожет применяться на указанном отрезке времени.
   */
   public void checkIfDatesCorrect(LocalDate startOfPeriod, LocalDate endOfPeriod) {
      boolean isStartOfPeriodCorrect = startOfPeriod.isAfter(start) || startOfPeriod.isEqual(start);
      boolean isEndOfPeriodCorrect = endOfPeriod.isBefore(end) || endOfPeriod.isEqual(end);
      if(!isStartOfPeriodCorrect || !isEndOfPeriodCorrect) {
         throw new IllegalArgumentException("Days not correct.");
      }
   }
}
```
Мне кажется, здесь можно сделать новый тип данных Period, в котором инкапсулировать проверку, что введенная дата находится внутри промежутка времени.
Вычисления в методе checkIfDatesCorrect так не будут выглядеть запутанными.
```

public class Period {

   LocalDate start;
   LocalDate end;
   
   public boolean isDateBelongs(LocalDate date) {
      return (date.isAfter(start) || date.isEqual(start)) &&  (date.isBefore(end) || date.isEqual(end));
   }
   
   public boolean isPeriodPelongs(LocalDate start, LocalDate end) {
      return this.isDateBelongs(start) && this.isDateBelongs(end);
   }
}
public class IncomeCalculator {

   Period period;
   
   /**
    Метод используется для проверки того, что данный калькулятор иожет применяться на указанном отрезке времени.
   */
   public void checkIfDatesCorrect(LocalDate startOfPeriod, LocalDate endOfPeriod) {
      if(period.isPeriodPelongs(startOfPeriod, endOfPeriod)) {
         return;
      }
         throw new IllegalArgumentException("Days not correct.");
   }
}
```



2. Примеры погрешности, которые чрезмерно "сужают" логику кода.
   
В примере ниже метод преобразует полученное сообщение класса Message в сущность класса Asset.
```
pubic boolean createAssetFromMessage(Message payload) {
    validateSource(payload);  
    Asset asset = new Asset();
    ...
    asset.setLabel(payload.getSource().split("\\.")[1]);
}

private void validateSource(Message payload) {
   if(payload.getSource().startsWith("PT.")) {
      return;
   }
   throw new IllegalArgumentException("Unknown source");
}
```
Message содержит поле source, который представляет собой строку в формате "PT.xxx", где xxx - это label, 
который надо записать в поле label класса Asset.
Здесь даже 2 неточности: 
1. source может приходит в любом формате. Например, с обновлением системы оно будет приходить в формате PT.A.xxx
2. Может добавиться любой другой префикс.  
Новый вариант:

```java

public boolean createAssetFromMessage(Message payload) {
    Asset asset = new Asset();
    ...
    String label = MessageSourceMask.getLabelFromSource(payload.getSource());
    asset.setLabel(label);
}

/**
 * Класс содержит допустимые форматы для поля source класса Message.
 * mask - префикс
 * extractor - функция для извлечения лейбла шаблону из строки source
 * При попытке извлечь лейбл из строки недопустимого формата выбрасывается исключение.
 */
public enum MessageSourceMask {
   PT("PT.A.", s -> s.split("\.")[2]),
   PT("PT.", s -> s.split("\.")[1]),
   XRC("XRC.", s -> s.split("\.")[1]);
   
   private String mask;
   private UnaryOperator<String> extractor;
    
    public static Optional<MessageSourceMask> fromSource(String source) {
        return Arrays.stream(values())
                .filter(s -> source.startsWith(s))
                .findFirst()
                .orElseThrow(()-> new IllegalArgumentException("Unknown source."));
    }
    
    public static String getLabelFromSource(String source) {
        return fromSource(source).map(m -> extractor.apply(source));
    }
   
}
```

3. Пример, когда интерфейс точно не должен быть меньше реализации.

