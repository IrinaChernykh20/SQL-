# Итоговая работа по SQL
1. Выведите название самолетов, которые имеют менее 50 посадочных мест?


select a.model 
from aircrafts a 
join seats s on a.aircraft_code=s.aircraft_code
group by a.model 
having count(fare_conditions)<50



2. Выведите процентное изменение ежемесячной суммы бронирования билетов, округленной до сотых.


with recursive r as (
	select min(date_trunc('month', book_date)) дата 
	from bookings b 
	union
	select дата + interval '1 month'
	from r 
	where дата < (select max(date_trunc('month', book_date)) from bookings))
	select дата ,b.sum,
	   round((((b.sum - lag(b.sum) over (order by дата))/lag(b.sum) over (order by дата)))*100,2) изменение  
	from r
	left join(
	select date_trunc('month', book_date), sum(total_amount)
	from bookings b 
	group by date_trunc('month', book_date)) b on b.date_trunc = дата
order by 1



3. Выведите названия самолетов не имеющих бизнес - класс. Решение должно быть через функцию array_agg.

select  model
from seats s 
left join aircrafts a on a.aircraft_code =s.aircraft_code 
group by a.aircraft_code 
having array_position(array_agg(fare_conditions), 'Business') is null

 
 

4. Найдите процентное соотношение перелетов по маршрутам от общего количества перелетов.
 Выведите в результат названия аэропортов и процентное отношение.
 Решение должно быть через оконную функцию.
 
 
select  a1.airport_name, a2.airport_name, round ((count (flight_id) /sum(count(flight_id))over() * 100),2)
from flights f 
join airports a1 on f.departure_airport = a1.airport_code 
join airports a2 on f.arrival_airport = a2.airport_code 
group by  a1.airport_code, a2.airport_code 


5. Выведите количество пассажиров по каждому коду сотового оператора, если учесть, что код оператора - это три символа после +7


select substring((contact_data ->>'phone'),3,3), count(passenger_id)
from tickets t 
group by substring((contact_data ->>'phone'),3,3)


6. Классифицируйте финансовые обороты (сумма стоимости перелетов) по маршрутам:
 До 50 млн - low
 От 50 млн включительно до 150 млн - middle
 От 150 млн включительно - high

 
select alias_for_case,count(*)
from 
 (select f.departure_airport,f.arrival_airport,sum(amount),
   case 
   	when sum(amount) < 50000000 then 'low'
  when sum(amount) >= 50000000 and sum(amount) < 150000000 then 'middle'
    when sum(amount) >= 150000000 then  'high'
   end alias_for_case
   from flights f 
join ticket_flights tf on f.flight_id = tf.flight_id
group by f.departure_airport,f.arrival_airport)e
group by alias_for_case



 7. Вычислите медиану стоимости перелетов,
 медиану размера бронирования и отношение медианы бронирования к медиане стоимости перелетов, округленной до сотых

 
with cte_5 as( 
select  percentile_cont(0.5) within group (order by total_amount) as "median_1"
from bookings b),
cte_6 as(
select percentile_cont(0.5) within group (order by amount)as "median_2"
from ticket_flights tf)
select  *, round((median_1::numeric / median_2::numeric ),2)
from cte_5,cte_6

8. Найдите значение минимальной стоимости полета 1 км для пассажиров.
То есть нужно найти расстояние между аэропортами и с учетом стоимости перелетов получить искомый результат
  Для поиска расстояния между двумя точками на поверхности Земли используется модуль earthdistance.
  Для работы модуля earthdistance необходимо предварительно установить модуль cube.
  Установка модулей происходит через команду: create extension название_модуля.

WITH cte_8 AS (
    SELECT 
        f.flight_id,
        (
            SELECT earth_distance(ll_to_earth(a1.latitude, a1.longitude), ll_to_earth(a2.latitude, a2.longitude)) / 1000
            FROM airports a1
            JOIN airports a2 ON f.arrival_airport = a2.airport_code
            WHERE f.departure_airport = a1.airport_code
        )  distance,
        min(amount) AS price
    FROM flights f
    JOIN ticket_flights tf ON f.flight_id = tf.flight_id
    GROUP BY f.flight_id 
)
SELECT 
    flight_id,
    distance,
    price,
    price / distance AS "минимальная Стоимость 1 км"
FROM cte_8
ORDER BY "минимальная Стоимость 1 км" ASC
LIMIT 1