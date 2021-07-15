
# SQL на практике

> P.S. в статье запросы написаны на postgresql

# Итак, для начала ознакомимся с предметной областью

У нас есть маленький магазичник, но довольно проходной, потому что торгует всем, начиная от **ПИВА** и заканчивая **ВОДКОЙ**…

![](/images/sql_lesson:art_1.png)

Владельцу понадобилось узнать, сколько людей заходит в магазин и в какое время, чтобы скоординировать смены кассиров и поставки. Во время максимальной посещаемости нужен выход дополнительного кассира, а во время минимальной можно заниматься залом и  раскладывать поступивший товар.

Для учета количества посетителей на магнитные рамки у входа было установлено **2 лазерных датчика.**

![](/images/sql_lesson:art_2.png)

Данные датчики при каждом срабатывании пишут соотвествующие данные в таблицу **sensor_triggers**, где **sensor_id** — **идентификтор датчика**, а **created_at** — **время срабатывания**.

![Схема таблицы](/images/sql_lesson:table.png)*Схема таблицы*

У датчика #1 — **sensor_id = 1**, а у датчика #2 — **sensor_id = 2**

По очередности срабатывания датчиков можно сделать вывод, вошел человек в магазин или вышел.

## Данные в таблице выглядят так:

![](/images/sql_lesson:table_data.png)

Записи #3, #4, #5 — **Лжесрабатывания**, потому что нарушена очередность срабатывания датчиков, и на основании этих данных нельзя сделать вывод, вошел человек или вышел.

## Возможные причины лжесрабатываний

![](/images/sql_lesson:risk_factors.png)

------

# C предметной областью мы закончили, теперь переходим к отчету.

Для начала, **уточняем** у заказчика что он хочет видеть, по возможности просим предоставить пример. Нам повезло, владелец показал, что он хочет видеть.

**“Отчет по проходимости магазина с почасовой группировкой”**

![](/images/sql_lesson:report_sample.png)

**Выводы из примера:**
1) В отчете должны отображаться промежутки с нулевой посещаемостью;
2) В отчете нужен итог за день;
3) Возможна погрешность в данных на вход и выход.

## Получается, что нам необходимо:

1. Из-за того, что в отчете необходимо показывать промежутки времени без посетителей надо сгенерировать возможные промежутки времени, чтобы к ним JOIN'ить данные по входу и выходу;

2. Обработать данные из таблицы **sensor_triggers**, чтобы в них прослеживалось количество входящих и выходящих людей, а также надо отбросить ложные срабатывания;

3. Добавить строку с итогами.

----

# Написание SQL запроса

## Примерная структура псевдозапроса

Полезно в виде черновика или наброска продумать структуру будущего запроса, тогда можно будет определиться в количестве запросов/подзапросов/join'ов и общей сложности.

```SQL
    ;WITH hours as (
      ...
    ),
    incomes as (
      SELECT * 
      FROM sensor_triggers
      WHERE ...
      GROUP BY hour(created_at)
    ),
    outcomes as (
      SELECT * 
      FROM sensor_triggers
      WHERE ...
      GROUP BY hour(created_at)
    ),
    rows as (
      SELECT *
      FROM hours
      LEFT JOIN incomes on incomes.hour = hours.hour
      LEFT JOIN outcomes on outcomes.hour = hours.hour
    ),
    final_row as (
      SELECT 
        'Итог' literal
        sum(rows.incomes),
        sum(rows.outcomes)
      FROM rows
    )
    SELECT * FROM rows
    UNION
    select * FROM final_row
```

* **hours** — временные интервалы
* **incomes** — количество вошедших людей
* **outcomes** — количество вышедших людей
* **rows** — данные сгруппированные по часам
* **final_rows** — агрегированные **rows**


## Генерируем промежутки времени

Не будем особо заморачиваться с автоматической генерацией строк, просто перечислим возможные варианты. Для генерации можно использовать рекурсивный WITH **(WITH RECURSIVE)**.

```SQL
    ;WITH hours as (
      SELECT 0 hour_id, '00:00 - 01:00' hour_literal UNION
      SELECT 1 hour_id, '01:00 - 02:00' hour_literal UNION
      SELECT 2 hour_id, '02:00 - 03:00' hour_literal UNION
      SELECT 3 hour_id, '03:00 - 04:00' hour_literal UNION
      SELECT 4 hour_id, '04:00 - 05:00' hour_literal UNION
      SELECT 5 hour_id, '05:00 - 06:00' hour_literal UNION
      SELECT 6 hour_id, '06:00 - 07:00' hour_literal UNION
      SELECT 7 hour_id, '07:00 - 08:00' hour_literal UNION
      SELECT 8 hour_id, '08:00 - 09:00' hour_literal UNION
      SELECT 9 hour_id, '09:00 - 10:00' hour_literal UNION
      SELECT 10 hour_id, '10:00 - 11:00' hour_literal UNION
      SELECT 11 hour_id, '11:00 - 12:00' hour_literal UNION
      SELECT 12 hour_id, '12:00 - 13:00' hour_literal UNION
      SELECT 13 hour_id, '13:00 - 14:00' hour_literal UNION
      SELECT 14 hour_id, '14:00 - 15:00' hour_literal UNION
      SELECT 15 hour_id, '15:00 - 16:00' hour_literal UNION
      SELECT 16 hour_id, '16:00 - 17:00' hour_literal UNION
      SELECT 17 hour_id, '17:00 - 18:00' hour_literal UNION
      SELECT 18 hour_id, '18:00 - 19:00' hour_literal UNION
      SELECT 19 hour_id, '19:00 - 20:00' hour_literal UNION
      SELECT 20 hour_id, '20:00 - 21:00' hour_literal UNION
      SELECT 21 hour_id, '21:00 - 22:00' hour_literal UNION
      SELECT 22 hour_id, '22:00 - 23:00' hour_literal UNION
      SELECT 23 hour_id, '23:00 - 00:00' hour_literal
      ORDER BY 1
    )
```
 **hour_literal** соответсвтует первому столбцу отчета, а **hour_id** необходим для соединения с данными о входе и выходе посетителей.

## Обрабатываем данные таблицы **sensor_triggers**

1. Данные необходимо сгруппировать по часам;

1. В данном случае проще не отбрасывать **лжесрабатывания**, а определить набор правил, когда засчитывать срабатывания как вход или выход. Будем считать, что человек с минимально возможной скоростью передвижения задействует два датчика за 5 секунд, тогда нам остается выбрать из таблицы последовательно задействованные датчики с максимальным периодом между срабатываниями = 5с. Тогда нам достаточно найти запись одного датчика, и второго с указанной разницей во времени.

## **Создаем incomes и outcomes**
```SQL
    ;WITH incomes as (
      -- группируем по часу, сразу считаем количество проходов
      SELECT date_part as , count(*) 
      FROM
        (SELECT
           DATE_PART('hour', started.created_at) date_part, 
           ROW_NUMBER() OVER (PARTITION BY started.id) rn    
           FROM sensor_triggers started
           JOIN sensor_triggers ended 
             -- Проверяем, что срабатывание между датчиками < 5 секунд
             ON ended.created_at - started.created_at BETWEEN '00:00:00'::time AND '00:00:05'::time
             -- Определяем, от какого к какому сенсерому ведем отсчет
             AND started.sensor_id = 1 AND ended.sensor_id = 2
             -- В данном случае считаем количество "приходящих" 
             -- посетителей (от датчика 1 к датчику 2)
        ) as temporally
        -- Если у датчика 2 были лжесрабатывания, то при джойне 
        -- будут лишние записи, для этого, поэтому мы ввели подсчет 
        -- номера строки в группе, т.к. нам достаточно получить 
        -- первую запись в группировке
        WHERE rn = 1 
        GROUP BY date_part 
    )
    SELECT * FROM incomes;
```

## **outcomes** выглядит абсолютно так же, только меняется порядок датчиков.
----
## Собираем отчет из всех подготовленных данных

```SQL
    ;WITH hours as (
      SELECT 0 hour_id, '00:00 - 01:00' hour_literal UNION
      SELECT 1 hour_id, '01:00 - 02:00' hour_literal UNION
      SELECT 2 hour_id, '02:00 - 03:00' hour_literal UNION
      SELECT 3 hour_id, '03:00 - 04:00' hour_literal UNION
      SELECT 4 hour_id, '04:00 - 05:00' hour_literal UNION
      SELECT 5 hour_id, '05:00 - 06:00' hour_literal UNION
      SELECT 6 hour_id, '06:00 - 07:00' hour_literal UNION
      SELECT 7 hour_id, '07:00 - 08:00' hour_literal UNION
      SELECT 8 hour_id, '08:00 - 09:00' hour_literal UNION
      SELECT 9 hour_id, '09:00 - 10:00' hour_literal UNION
      SELECT 10 hour_id, '10:00 - 11:00' hour_literal UNION
      SELECT 11 hour_id, '11:00 - 12:00' hour_literal UNION
      SELECT 12 hour_id, '12:00 - 13:00' hour_literal UNION
      SELECT 13 hour_id, '13:00 - 14:00' hour_literal UNION
      SELECT 14 hour_id, '14:00 - 15:00' hour_literal UNION
      SELECT 15 hour_id, '15:00 - 16:00' hour_literal UNION
      SELECT 16 hour_id, '16:00 - 17:00' hour_literal UNION
      SELECT 17 hour_id, '17:00 - 18:00' hour_literal UNION
      SELECT 18 hour_id, '18:00 - 19:00' hour_literal UNION
      SELECT 19 hour_id, '19:00 - 20:00' hour_literal UNION
      SELECT 20 hour_id, '20:00 - 21:00' hour_literal UNION
      SELECT 21 hour_id, '21:00 - 22:00' hour_literal UNION
      SELECT 22 hour_id, '22:00 - 23:00' hour_literal UNION
      SELECT 23 hour_id, '23:00 - 00:00' hour_literal
      ORDER BY 1
    ),
    incomes as (
      SELECT date_part, count(*) 
      FROM
        (
          SELECT
            DATE_PART('hour', started.created_at) date_part,
            ROW_NUMBER() OVER (PARTITION BY started.id) rn
          FROM sensor_triggers started
          JOIN sensor_triggers ended 
            ON ended.created_at - started.created_at BETWEEN '00:00:00'::time AND '00:00:05'::time
            AND started.sensor_id = 1 AND ended.sensor_id = 2
            -- Параметр "дата" формирования отчета
            AND started.created_at::date = '2019.06.09'
        ) as temporally
      WHERE rn = 1 
      GROUP BY date_part 
    ),
    outcomes as (
      SELECT date_part, count(*) 
      FROM
        (
          SELECT
            DATE_PART('hour', started.created_at) date_part, 
            ROW_NUMBER() OVER (PARTITION BY started.id) rn
          FROM sensor_triggers started
          JOIN sensor_triggers ended 
            ON ended.created_at - started.created_at BETWEEN '00:00:00'::time AND '00:00:05'::time
            AND started.sensor_id = 2 AND ended.sensor_id = 1
            -- Параметр "дата" формирования отчета
            AND started.created_at::date = '2019.06.09'
        ) as temporally
      WHERE rn = 1 
      GROUP BY date_part 
    ),
    rows as (
      SELECT 
        hours.hour_literal,
        coalesce(incomes.count, 0) income,
        coalesce(outcomes.count, 0) outcome
      FROM hours
      LEFT JOIN outcomes on hours.hour_id = outcomes.date_part
      LEFT JOIN incomes on hours.hour_id = incomes.date_part
    ),
    final_row as (
      SELECT 'Итог', sum(income), sum(outcome) from rows
    )
    SELECT * FROM rows
    UNION ALL
    SELECT * from final_row
```

# Итог

В общем-то на этом и все, в конце используется **UNION ALL** чтобы добавить строку в конец выборки, т.к. **UNION ALL** просто соединяет две выборки в указанном порядке, в отличии от **UNION**, который объединяет выборки и удаляет дубликаты (для поиска дубликатов выборка сортируется).

**Спасибо, что прочитали до конца! Надеюсь, что статья научит чему-нибудь новому :)**