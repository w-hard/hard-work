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


3. Когда интерфейс точно не должен быть меньше реализации.

В данном методе рассчитывается сумма дохода в зависимости от значения процентной ставки и даты начисления дохода. 

``` java
public BigDecimal calculateIncome(BigDecimal rate, LocalDate rateStartDate, LocalDate rateEndDate, LocalDate periodStart) {
        boolean containFirstDayOfPeriod = rateStartDate.equals(periodStart);
        long daysBetween = ChronoUnits.DAYS.between(rateStartDate, rateEndDate);
        long daysForCalc = hasFirstDayOfPeriod ? daysBetween - 1 + curve : daysBetween;
        BigDecimal rateWithSpread = rate.add(spread);

        return nominal.multiply(rateWithSpread).multiply(BigDecimal.valueOf(daysForCalc)).divide(BigDecimal.valueOf(365));
    }
```
Можно выделить новый тип InterestRate, который будет содержать период действия ставки и значение ставки.
Спецификация получается больше, чем реализация. 

```java

public class Rate {
        private Period period;
        private BigDecimal value;

        public Long getPeriodLenght() {
            return period.getLength();
        }
}
   /**
    * Сумма дохода рассчитывается по формуле:
    * Сумма = номинал * ("значение ставки" + спред) * "количество дней в расчетном периоде" / 365
    * Количество дней в расчетном периоде рассчитывается по формуле:
    * Дни = конец действия ставки - начало расчетного периода + срок кривой доходности
    * Срок кривой доходности равен curve, если дата начала действия ставки приходится на первый день расчетного периода, в противном случае равен 0
    */
   public BigDecimal calculateIncome(InterestRate rate, LocalDate billingPeriodStart) {
      long periodLength = rate.getPeriodLenght();
      long daysForCalc = rate.getPeriod().isStartOfPeriod(billingPeriodStart) ? periodLength - 1 + curve : periodLength;
      BigDecimal rateWithSpread = rate.getValue().add(spread);

      return nominal.multiply(rateWithSpread).multiply(BigDecimal.valueOf(daysForCalc)).divide(BigDecimal.valueOf(365));
   }
```



