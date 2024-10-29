# SQL и базы данных

Уверенные знания SQL.
Знаком с запросами на выборку и изменение денных.
Есть опыт использования CTE, оконных функциями, JOIN и пр.

Могу развернуть БД через DDL.
Уверенно владею `DBeaver`.

+ Примеры запросов (контекст — работа с учебной базой данных «Авиаперевозки» с [edu.postgrespro.ru](https://edu.postgrespro.ru/)): +
  
  ```sql
  -- Установка модулей
  
  create extension cube
  
  create extension earthdistance
  
  --1. Выведите название самолетов, которые имеют менее 50 посадочных мест?
  
  select  a.aircraft_code, a.model, count(s.seat_no) as "Количество_мест"
  from aircrafts a
  join seats s
  on a.aircraft_code = s.aircraft_code
  group by a.aircraft_code
  having COUNT(s.seat_no) < 50
  
  --2. Выведите процентное изменение ежемесячной суммы бронирования билетов, округленной до сотых.
  
  --Способ 1
  --explain analyze 45264.39/425.236
  with total_month_sum as (
      select extract(month from b.book_date) as "Месяц",
             extract(YEAR from b.book_date) as "Год",
             SUM(b.total_amount) as "Cумма_бронирования_за_месяц",
             LAG(SUM(b.total_amount)) over (order by extract(YEAR from b.book_date), extract(month   from b.book_date)) as "Предыдущая_сумма"
      from bookings b
      group by extract(MONTH from b.book_date), extract(YEAR from b.book_date)
  )
  select "Месяц",
         "Год",
         "Cумма_бронирования_за_месяц",
         coalesce("Предыдущая_сумма", 0) as "Предыдущая_сумма_",
         coalesce(ROUND((("Cумма_бронирования_за_месяц" - "Предыдущая_сумма") / "Предыдущая_сумма") *   100, 2), 0) AS "Процентное_изменение(%)"
  from total_month_sum
  order by "Год", "Месяц"
  
  
  --Способ 2, ложный запрос, если будет пропуск по какому-то месяцу?????
  --explain analyze 82578.28/289.136
  with recursive r as (
    select min(date_trunc('month', book_date)) x
    from bookings b
    union 
    select x + interval '1 month'
    from r 
    where x < (select max(date_trunc('month', book_date)) from bookings))
  select x::date as "Месяц", coalesce(sum, 0) as "Cумма_бронирования_за_месяц",
    lag(coalesce(sum, 0), 1, 0) over (order by x) as "Предыдущая_сумма",
    coalesce(round((coalesce(sum, 0) - lag(coalesce(sum, 0), 1, 0) over (order by x)) * 100.0 / nullif  (lag(coalesce(sum, 0), 1, 0) over (order by x), 0), 2), 0) as "Процентное изменение(%)"
  from r
  left join (
    select date_trunc('month', book_date), sum(total_amount)
    from bookings 
    group by 1) p on p.date_trunc = x
  order by 1
  
  
  --3. Выведите названия самолетов не имеющих бизнес - класс. Решение должно быть через функцию   array_agg.
  
  select a.aircraft_code, a.model, array_agg(distinct s.fare_conditions) as "Тип_билета"
  from aircrafts a 
  join seats s 
  on a.aircraft_code = s.aircraft_code
  group by a.aircraft_code
  having not 'Business' = any(array_agg(distinct s.fare_conditions))
  
  --4. Вывести накопительный итог количества мест в самолетах по каждому аэропорту на каждый день,   учитывая только те самолеты, которые летали пустыми и только те дни, где из одного аэропорта таких   самолетов вылетало более одного.
  --В результате должны быть код аэропорта, дата, количество пустых мест в самолете и накопительный   итог.
  
  --(не совсем понятно задание, поэтому 2 варианта вывода)
  --вариант 1
  
  with cte1 as (
  	SELECT f.departure_airport as "Аэропорт_вылета", date_trunc('day', f.actual_departure) as   "Дата_вылета", f.flight_no as "Номер_рейса", f.aircraft_code as "Код_самолета",
  	count(*) over(partition by f.departure_airport, date_trunc('day', f.actual_departure)) as rnnn
  	FROM flights f
  	WHERE f.flight_id NOT IN (SELECT DISTINCT flight_id FROM ticket_flights) and f.actual_departure   is not null),
  cte2 as (
  	select  a.aircraft_code as "Код2", a.model, count(s.seat_no) as "Количество_мест"
  	from aircrafts a
  	join seats s
  	on a.aircraft_code = s.aircraft_code
  	group by a.aircraft_code)
  select cte1."Аэропорт_вылета" as "Аэропорт_вылета", cte1."Дата_вылета" as "Дата_вылета", cte2.  "Количество_мест" as "Ко-во_мест_в_самолетах", 
  	sum (cte2."Количество_мест") over (partition by cte1."Аэропорт_вылета" order by cte1.  "Дата_вылета") as "Накоп_итог_мест_за_день"
  	from cte1
  	join cte2 on cte1."Код_самолета" = cte2."Код2"
  	where  cte1.rnnn > 1 
  	order by cte1."Аэропорт_вылета", cte1."Дата_вылета"
  
  --вариант 2
  	
  with cte1 as (
  	SELECT f.departure_airport as "Аэропорт_вылета", date_trunc('day', f.actual_departure) as   "Дата_вылета", f.flight_no as "Номер_рейса", f.aircraft_code as "Код_самолета",
  	count(*) over(partition by f.departure_airport, date_trunc('day', f.actual_departure)) as rnnn
  	FROM flights f
  	WHERE f.flight_id NOT IN (SELECT DISTINCT flight_id FROM ticket_flights) and f.actual_departure   is not null),
  cte2 as (
  	select  a.aircraft_code as "Код2", a.model, count(s.seat_no) as "Количество_мест"
  	from aircrafts a
  	join seats s
  	on a.aircraft_code = s.aircraft_code
  	group by a.aircraft_code)
  select cte1."Аэропорт_вылета" as "Аэропорт_вылета", cte1."Дата_вылета" as "Дата_вылета", cte2.  "Количество_мест" as "Ко-во_мест_в_самолетах", 
  	sum (cte2."Количество_мест") over (partition by cte1."Аэропорт_вылета" order by cte1.  "Дата_вылета", cte1."Номер_рейса") as "Накоп_итог_мест"
  	from cte1
  	join cte2 on cte1."Код_самолета" = cte2."Код2"
  	where  cte1.rnnn > 1 
  	order by cte1."Аэропорт_вылета", cte1."Дата_вылета"
  
  --5. Найдите процентное соотношение перелетов по маршрутам от общего количества перелетов. (аэропорт   вылета + прилета = маршрут 710 или 618 строк)
  -- Выведите в результат названия аэропортов и процентное отношение.
  -- Решение должно быть через оконную функцию.
  
  
  select airoports_fligts.departure_airport_name as "Аэропорт_вылета", 
  	airoports_fligts.arrival_airport_name as "Аэропорт_прилета",
  	airoports_fligts."количество_перелетов_по_маршруту",
  	airoports_fligts."Общее_количество_перелетов",
  	airoports_fligts."количество_перелетов_по_маршруту" * 100.0 / airoports_fligts.  "Общее_количество_перелетов" as "Процентное_соотношение"
  from(
  	select flight_id, flight_no, departure_airport_name, arrival_airport_name,
  	count(*) over (partition by flight_no order by flight_no) as "количество_перелетов_по_маршруту",
  	row_number () over (partition by flight_no order by flight_no) as r_n,
  	count(*) over () as "Общее_количество_перелетов"
  	from flights_v
  	where actual_departure is not null
  	order by flight_no) as airoports_fligts
  where airoports_fligts.r_n = 1
  
  
  --6. Выведите количество пассажиров по каждому коду сотового оператора, если учесть, что код   оператора - это три символа после +7
  
  --Способ 1
  
  --explain analyze 62576.13/440.557
  select SUBSTRING(contact_data->>'phone', 3, 3) AS "Код_оператора",
         COUNT(*) AS "Количество_пассажиров"
  from tickets t
  group by SUBSTRING(contact_data->>'phone', 3, 3)
  order by "Код_оператора"
  
  --сопсоб2
  --explain analyze 68077.13/469.128
  select subq."Код_оператора", subq."Количество_пассажиров"
  from 
  (select SUBSTRING(contact_data->>'phone', 3, 3) AS "Код_оператора",
         COUNT(*) over (partition by SUBSTRING(contact_data->>'phone', 3, 3)) AS   "Количество_пассажиров",
         row_number() over (partition by SUBSTRING(contact_data->>'phone', 3, 3))
  from tickets t) as subq
  where row_number = 1
  
  
  --7. Классифицируйте финансовые обороты (сумма стоимости перелетов) по маршрутам: (3 практика)
  -- До 50 млн - low
  -- От 50 млн включительно до 150 млн - middle
  -- От 150 млн включительно - high (25 маршрутов)
  -- Выведите в результат количество маршрутов в каждом полученном классе
  
  
  --Способ1:
  --explain analyze  20294.12/300.632
  
  with cte as (
    select 
      sum(tf.amount) as sum_class
    from routes r
    join flights f on r.departure_airport = f.departure_airport and r.arrival_airport = f.  arrival_airport
    join ticket_flights tf on f.flight_id = tf.flight_id 
    where f.actual_departure is not null
    group by f.flight_no
  )
  select "Класс_по_обороту", "Количество_маршр_в_классе"
   from (select 
    case 
      when c.sum_class < 50000000  then 'low' 
      when c.sum_class >= 50000000 and c.sum_class < 150000000 then 'middle' 
      when c.sum_class >= 150000000 then 'high' 
    end as "Класс_по_обороту",
    count(*) over (partition by 
      case 
        when c.sum_class < 50000000  then 'low' 
        when c.sum_class >= 50000000 and c.sum_class < 150000000 then 'middle' 
        when c.sum_class >= 150000000 then 'high' 
      end) as "Количество_маршр_в_классе",
     row_number () over (partition by 
      case 
        when c.sum_class < 50000000  then 'low' 
        when c.sum_class >= 50000000 and c.sum_class < 150000000 then 'middle' 
        when c.sum_class >= 150000000 then 'high' 
      end
      order by 
      case 
        when c.sum_class < 50000000  then 'low' 
        when c.sum_class >= 50000000 and c.sum_class < 150000000 then 'middle' 
        when c.sum_class >= 150000000 then 'high' 
      end) as rn
  from cte c) as subq
  where subq.rn = 1
  
  
  
  --Способ2:
  --explain analyze  77056.13/1857.854
  
  with cte as (select classif.sum_class,
  	case 
  		when classif.sum_class < 50000000  then 'low' 
  		when classif.sum_class >= 50000000 and classif.sum_class < 150000000 then 'middle' 
  		when classif.sum_class >= 150000000 then 'high' 
  	end group_case
  	from (
  		select subq.subqf_n, subq.subq_a, subq.summmm as sum_class
  		from (
  			select f.flight_no as subqf_n, tf.amount as subq_a,
  			row_number () over (partition by f.flight_no),
  			sum (tf.amount) over (partition by f.flight_no order by f.flight_no) as summmm
  			from routes r
  			join flights f on r.departure_airport = f.departure_airport and r.arrival_airport = f.  arrival_airport
  			join ticket_flights tf on f.flight_id = tf.flight_id 
  		where f.actual_departure is not null) as subq
  	where row_number = 1) as classif)
  select finall."Класс_по_обороту", finall."Количество_маршр_в_классе"
  from (
  	select cte.group_case as "Класс_по_обороту",
  	count (*) over (partition by cte.group_case) as "Количество_маршр_в_классе",
  	row_number () over (partition by cte.group_case order by cte.group_case) as final_subs
  	from cte) as finall
  where finall.final_subs = 1
  
  
  -- 8. Вычислите медиану стоимости перелетов, медиану размера бронирования и отношение медианы   бронирования к медиане стоимости перелетов, округленной до сотых (не более 2 таблиц)
  
  with total_amount_mediana as (
  	select
  	PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY b.total_amount) AS "Медиана_размера_бронирования"
  	from bookings b
  ),
  median_flight_cost as (
  	select
  	PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY tf.amount) AS "Медиана_cтоимости"
  	from ticket_flights tf 
  )
  select tam."Медиана_размера_бронирования", 
  	mfc."Медиана_cтоимости",
  	ROUND("Медиана_размера_бронирования"::numeric / "Медиана_cтоимости"::numeric, 2) as   "Отношение_МРБ/МС"
  from total_amount_mediana tam, median_flight_cost mfc
  
  
  --9. Найдите значение минимальной стоимости полета 1 км для пассажиров. То есть нужно найти   расстояние между аэропортами и с учетом стоимости перелетов получить искомый результат
  -- Для поиска расстояния между двумя точками на поверхности Земли используется модуль earthdistance. 
  --  Для работы модуля earthdistance необходимо предварительно установить модуль cube. 
  --  Установка модулей происходит через команду: create extension название_модуля. 
  
  
  with min_amount as (select ticket_no, flight_id, min(amount) as maa
  	from ticket_flights tf
  	group by ticket_no,  flight_id
  	order by min(amount)
  	limit 1
  ),
  a2 as (select departure_airport, arrival_airport, maa
  	from min_amount ma, flights f
  	where f.flight_id = ma.flight_id
  ),
  distance1 as (
  	select latitude, longitude
  	from a2, airports a 
  	where a.airport_code = a2.departure_airport
  ),
  distance2 as (
  	select latitude, longitude
  	from a2, airports ars 
  	where ars.airport_code = a2.arrival_airport
  )
  select min_amount.maa as "Минимальная_цена_билета",
  a2.departure_airport as "Аэропорт_вылета", 
  a2.arrival_airport as "Аэропорт_прилета",
  earth_distance(ll_to_earth(distance1.latitude, distance1.longitude), ll_to_earth(distance2.latitude,   distance2.longitude))/100 AS "Дистанция_км", 
  min_amount.maa/(earth_distance(ll_to_earth(distance1.latitude, distance1.longitude), ll_to_earth  (distance2.latitude, distance2.longitude))/100) as "Мин_ стоимость_1км(руб)" 
  from min_amount, a2, distance1, distance2
  
  ```

## Документирование баз данных и СУБД

Опыт создания статических справочников баз данных с помощью Foliant и препроцессора [DBDoc](https://foliant-docs.github.io/docs/preprocessors/dbdoc/) для работы с разными СУБД.

Опыт написания миграций для `Liquibase` и нативных `.sql`.

Есть опыт организации процессов создания документации для отечественной in-memory СУБД (ссылка на документацию — по запросу).
