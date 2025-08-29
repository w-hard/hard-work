Пример таблиц, которые можно денормализовать.

У нас проект нельзя назвать высоконагруженным, поэтому не смогла найти пример, когда таблицы можно денормализовать именно для повышения производительности.
Но нашла пример, когда возможно изначально не была нужна дополнительная таблица, и сейчас 2 таблицы можно денормализовать для удобства.

Таблицы:
1) _product_release_ - содержит выпуски новых финансовых продуктов.
Некоторые выпуски могут иметь одинаковую "базу" (основные параметры).
Но дополнительные параметры будут отличаться (например выпуски нацелены на разные типы инвесторов) поэтому это будут отдельные сущности.
2) _product_type_ - это "база". Содержит параметры, общие для нескольких выпусков.
3) _product_type_params_ - содержит допллнительные параметры выпуска. Эти параметры незафиксированы, и новые параметры могут добавляться для новых выпусков.


Псевдо ddl таблиц:
```
CREATE TABLE product_release (
  code,
  status,
  product_type_id , -- foreign key product_type(id) ....
);

CREATE TABLE product_type (
  "name",
  asset_id, -- foreign key to asset(id)
  source_id , -- foreign key to source(id);
  product_release_type
);

CREATE TABLE product_type_params (
  param_name,
  param_value,
  product_type_id -- foreign key product_type(id)
);
```

Почему можно денормализовать:
1) Фактически, по бизнес-логике, когда мы работаем с product_type, мы всегда также получаем его параметры.
2) product_type - это не самостоятельная бизнес-сущность. По сути эта таблица была создана, чтобы выделить некоторые общие параметры для разных product_release.
Поэтому кажется, что можно внедрить product_type в product_type_params. И наверное, если бы переименовать полученную таблицу в product_release_params, то это бы лучше отражало смысл.

```
CREATE TABLE product_release (
  code,
  status,
  product_release_type,
  ...
);
CREATE TABLE product_release_params (
  source_id,
  asset_id,
  param_name,
  param_value,
  product_release -- foreign key product_release(id)
);
```
