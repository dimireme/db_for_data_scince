# db_for_data_scince

Домашние работы по курсу "Базы данных для аналитиков" от GeekBrains.

<details>
  <summary>Урок 1. Аналитика в бизнес-задачах.</summary>

1.  Залить в свою БД данные по продажам. Часть таблицы orders в csv, исходник [здесь](https://drive.google.com/drive/folders/1C3HqIJcABblKM2tz8vPGiXTFT7MisrML?usp=sharing).

2.  Проанализировать, какой период данных выгружен.

    ```sql
    SELECT min(o_date), max(o_date) FROM orders;
    ```

    | min(o_date) | max(o_date) |
    | ----------- | ----------- |
    | 2016-01-01  | 2017-12-31  |

3.  Посчитать количество строк, заказов и уникальных пользователей, которые совершали заказы.

    ```sql
    SELECT
     count(id_o) AS total,
     count(DISTINCT id_o) AS unique_orders,
     count(DISTINCT user_id) AS unique_users
    FROM orders;
    ```

    | total   | unique_orders | unique_users |
    | ------- | ------------- | ------------ |
    | 2002804 | 2002804       | 1015119      |

4.  По годам посчитать средний чек, среднее количество заказов на пользователя, сделать вывод, как изменялись эти показатели год от года.

    ```sql
    SELECT
      YEAR(o_date) AS 'year',
      round(avg(price), 0) AS avg_price,
      count(id_o) / count(DISTINCT user_id) AS avg_orders
    FROM orders
    GROUP BY YEAR(o_date);
    ```

    | year | avg_price | avg_orders |
    | ---- | --------- | ---------- |
    | 2016 | 2096      | 1.9352     |
    | 2017 | 2398      | 1.7430     |

5.  Найти количество пользователей, которые покупали в одном году и перестали покупать в следующем.

    ```sql
    SELECT count(t16.user_id) AS 'count' FROM
      (SELECT DISTINCT user_id FROM orders WHERE YEAR(o_date) = 2016) t16
    LEFT JOIN
      (SELECT DISTINCT user_id FROM orders WHERE YEAR(o_date) = 2017) t17
    ON t16.user_id = t17.user_id
    WHERE t17.user_id IS NULL;
    ```

    | count  |
    | ------ |
    | 360225 |

6.  Найти ID самого активного по количеству покупок пользователя.

    ```sql
    SELECT
      user_id,
      count(id_o) AS orders
    FROM orders
    GROUP BY user_id
    ORDER BY orders DESC LIMIT 1;
    ```

    | user_id | orders |
    | ------- | ------ |
    | 765861  | 3183   |

</details>

<details>
  <summary>Урок 3. Типовые методы анализа данных. RFM-анализ.</summary>

Главная задача: сделать RFM-анализ на основе данных по продажам за 2 года.

1.  Определяем критерии для каждой буквы R, F, M (т.е. к примеру, R=3 для клиентов, которые покупали <= 30 дней от последней даты в базе, R=2 для клиентов, которые покупали > 30 и менее 60 дней от последней даты в базе и т.д.).

| номер | r               | f                | m                   |
| ----- | --------------- | ---------------- | ------------------- |
| 1     | 60 < days       | 20 <= period     | spend < 1000        |
| 2     | 30 < days <= 60 | 10 <= period <20 | 1000 <= spend <5000 |
| 3     | days <= 30      | period < 10      | 5000 <= spend       |

При этом, если пользователь совершил менее 4-х покупок, при определении периода f, он попадёт в категорию 1.

2.  Для каждого пользователя получаем набор из 3 цифр (от 111 до 333, где 333 – самые классные пользователи)

    ```sql
    DROP TABLE IF EXISTS rfm_analys;

    CREATE TABLE rfm_analys
    SELECT
      user_id,
      min(o_date) AS first_activity,
      max(o_date) AS last_activity,
      count(id_o) AS orders_count,
      sum(price) AS total_price,
      CASE
        WHEN count(id_o) < 4 THEN "1"
        ELSE (
          CASE
            WHEN (TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) / (count(id_o) - 1)) < 10 THEN "3"
            WHEN (TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) / (count(id_o) - 1)) >= 10 AND (TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) / (count(id_o) - 1)) < 20 THEN "2"
            ELSE "1" END
        ) END AS f,
      CASE
        WHEN sum(price) < 1000 THEN "1"
        WHEN sum(price) >= 1000 AND sum(price) < 5000 THEN "2"
        ELSE "3" END AS m,
      CASE
        WHEN TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) >= 0 AND TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) < 30 THEN "1"
        WHEN TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) >= 30 AND TIMESTAMPDIFF(DAY, max(o_date), date('2018-01-01')) < 60 THEN "2"
        ELSE "3" END AS r
    FROM orders
    GROUP BY user_id;
    ```

3.  Вводим группировку, к примеру, 333 и 233 – это Vip, 1XX – это Lost, остальные Regular ( можете ввести боле глубокую сегментацию)

    ```sql
    SELECT
      count(user_id) AS count_users,
      sum(total_price) AS sum_price,
      r,
      f,
      m
    FROM rfm_analys
    GROUP BY r, f, m;
    ```

<details>
  <summary>результат</summary>

| count_users | sum_price      | r   | f   | m   |
| ----------- | -------------- | --- | --- | --- |
| 24396       | 15168338.500   | 1   | 1   | 1   |
| 53682       | 125086946.600  | 1   | 1   | 2   |
| 20243       | 330105237.000  | 1   | 1   | 3   |
| 44          | 159354.300     | 1   | 2   | 2   |
| 1070        | 72138007.200   | 1   | 2   | 3   |
| 1           | 842.800        | 1   | 3   | 1   |
| 68          | 243700.100     | 1   | 3   | 2   |
| 489         | 176602634.250  | 1   | 3   | 3   |
| 18771       | 11539929.100   | 2   | 1   | 1   |
| 41026       | 96279822.800   | 2   | 1   | 2   |
| 17085       | 256956597.100  | 2   | 1   | 3   |
| 24          | 88804.100      | 2   | 2   | 2   |
| 503         | 26589229.800   | 2   | 2   | 3   |
| 2           | 1253.000       | 2   | 3   | 1   |
| 39          | 134100.400     | 2   | 3   | 2   |
| 162         | 18564756.000   | 2   | 3   | 3   |
| 243762      | 143867896.900  | 3   | 1   | 1   |
| 453734      | 1039986626.700 | 3   | 1   | 2   |
| 133628      | 1605575577.900 | 3   | 1   | 3   |
| 18          | 13876.100      | 3   | 2   | 1   |
| 623         | 2081179.100    | 3   | 2   | 2   |
| 3271        | 82875090.200   | 3   | 2   | 3   |
| 76          | 43045.100      | 3   | 3   | 1   |
| 735         | 2239603.800    | 3   | 3   | 2   |
| 1667        | 64766158.800   | 3   | 3   | 3   |

</details>

Всего пользователей и потраченных ими денег:

```sql
SELECT count(user_id), sum(total_price) FROM rfm_analys;
```

| count(user_id) | sum(total_price) |
| -------------- | ---------------- |
| 1015119        | 4071108607.650   |

Добавим категории пользователей.

```sql
ALTER TABLE rfm_analys ADD category VARCHAR(10);
UPDATE rfm_analys set category = (
  CASE
    WHEN (r='3' OR r='2') AND f = '3' AND m='3' THEN 'vip'
    WHEN r='1'	THEN 'lost'
    ELSE 'regular' END
);
```

4.  Для каждой группы из п. 3 находим количество пользователей, которые попали в них и % товарооборота, которое они сделали на эти 2 года.

    ```sql
    SELECT
      sum(total_price) AS total_spend,
      concat(round(( sum(total_price)/ (SELECT sum(total_price) FROM rfm_analys) * 100 ),2),'%') AS percentage,
      count(user_id) AS users_count,
      category
    FROM rfm_analys
    GROUP BY category
    ORDER BY total_spend DESC;
    ```

    | total_spend    | percentage | users_count | category |
    | -------------- | ---------- | ----------- | -------- |
    | 3268272632.100 | 80.28%     | 913297      | regular  |
    | 719505060.750  | 17.67%     | 99993       | lost     |
    | 83330914.800   | 2.05%      | 1829        | vip      |

5.  Проверяем, что общее кол-во пользователей бьется с суммой кол-ва пользователей по группам из п. 3 (если у вас есть логические ошибки в создании групп, у вас не собьются цифры). То же самое делаем и по деньгам.

    Количество пользователей в пункте 4 `98102 + 8085 + 180 = 106367` совпадает с количеством пользователей в пункте 3.

    Количество потраченных денег в пункте 4 `265941536.70 + 27605275.60 + 7035198.80 = 300582011.1` совпадает со значением в пункте 3.

</details>

<details>
  <summary>Урок 4. Типовая аналитика маркетинговой активности. Кагортный анализ.</summary>

На основе данных по продажам за 16 и 17 год на основе когортного анализа по ГГММ первой покупки спрогнозировать товарооборот января 2018 года (с выводом кэфов поведения когротны по порядковому номеру месяца). Т.е. строим все когорты, понимаем как вымирает когорта. После 14 месяца обычно начинает мерцание на 2-5 процентов от первоначальной суммы. Итого, мы знаем как в среднем живут когорты, строим прогноз на один месяц для уже существующих когорт и предполагаем какой сформируется новая.

Запрос данных для разбивки по кагортам:

```sql
SELECT
  c.cogort,
  date_format((o.o_date), "%Y-%m") AS purchase_date,
  sum(o.price) AS revenue
FROM orders o
JOIN (
  SELECT
    user_id,
    date_format(min(o_date), "%Y-%m") AS cogort
  FROM orders
  GROUP BY user_id
) c
ON o.user_id = c.user_id
GROUP BY c.cogort, date_format((o.o_date), "%Y-%m");
```

Последние 10 строк результата:

| cogort  | purchase_date | revenue       |
| ------- | ------------- | ------------- |
| 2017-09 | 2017-09       | 114721028.800 |
| 2017-09 | 2017-10       | 5214909.700   |
| 2017-09 | 2017-11       | 4504822.000   |
| 2017-09 | 2017-12       | 3960622.400   |
| 2017-10 | 2017-10       | 138653454.800 |
| 2017-10 | 2017-11       | 6344545.200   |
| 2017-10 | 2017-12       | 5199659.500   |
| 2017-11 | 2017-11       | 163478573.300 |
| 2017-11 | 2017-12       | 6732426.400   |
| 2017-12 | 2017-12       | 191394529.900 |

Результаты расчета в файле lesson_4_cagort.csv

Получили распределение по кагортам:

| cogort  | покупка в первом месяце | коэффициент | прогноз      |
| ------- | ----------------------- | ----------- | ------------ |
| 2016-01 | 112520331.35            | 14.64%      | 16470434.68  |
| 2016-02 | 76659972.9              | 7.90%       | 6058704.63   |
| 2016-03 | 89331704                | 6.53%       | 5835736.9    |
| 2016-04 | 87505127.5              | 5.10%       | 4460832.7    |
| 2016-05 | 77422482.9              | 4.61%       | 3567982.48   |
| 2016-06 | 68918992.1              | 4.21%       | 2902944.94   |
| 2016-07 | 71512003.5              | 4.22%       | 3017073.15   |
| 2016-08 | 83235113.5              | 3.92%       | 3259241.07   |
| 2016-09 | 84694696.8              | 4.09%       | 3462294.03   |
| 2016-10 | 106447878.6             | 3.29%       | 3506519.8    |
| 2016-11 | 126087879.4             | 3.18%       | 4004945      |
| 2016-12 | 127987883.1             | 3.44%       | 4407360.63   |
| 2017-01 | 123985677.2             | 10.82%      | 13412565.81  |
| 2017-02 | 104769212.1             | 6.42%       | 6728552.35   |
| 2017-03 | 118447847.7             | 5.30%       | 6275869.32   |
| 2017-04 | 109542408.5             | 4.42%       | 4843086.94   |
| 2017-05 | 129331934.2             | 4.18%       | 5412404.33   |
| 2017-06 | 110123214.6             | 4.23%       | 4653765.93   |
| 2017-07 | 113386903               | 3.53%       | 4002233.86   |
| 2017-08 | 117063009.7             | 3.27%       | 3831333.23   |
| 2017-09 | 114721028.8             | 2.62%       | 3008166.84   |
| 2017-10 | 138653454.8             | 2.53%       | 3509525.1    |
| 2017-11 | 163478573.3             | 1.95%       | 3188852.44   |
| 2017-12 | 191394529.9             | 4.56%       | 8725097.76   |
| 2018-01 | 118253004.28            | 100.00%     | 118253004.28 |

Суммарная прибыль в январе 2018 года составит 246 798 528.19 рублей.

Для кагорт 2016-01 - 2016-11 коэффициент рассчитывался как средний процент покупки от покупки в первом месяце за период с 15 по последующие месяцы с месяца первой покупки.

Для кагорт 2016-12 - 2017-11 коэффициент рассчитывался как среднее от скорости затухания покупок за первые 14 месяцев.

Для кагорты 2017-12 ожидаенмый процент покупок в январе составил 91% от первой покупки, поэтому для этой кагорты был взят процент покупок кагортой 2016-12 в январе 2017 по отношению к покупкам в первом месяце (декабрь 2016). Он составил 4.56%.

Для ожидаемой кагорты 2018-01 объём затрат рассчитывался как средние затраты кагорт 2016-01 и 2017-01 в первом месяце покупки (январь 2016 и январь 2017 соответственно).

</details>

<details>
  <summary>Урок 5. Системы web-аналитики. Прогноз по категориям пользователей.</summary>

В качестве ДЗ делам прогноз ТО на 12.2017. В качестве метода прогноза - считаем сколько денег тратят группы клиентов вдень.

1.  Группа часто покупающих и которые последний раз покупали не так давно. Считаем сколько денег оформленного заказа приходится на 1 день. Умножаем на 30.

Определим среднее число покупок у пользователей

```sql
select count(id_o) / count(DISTINCT user_id) as average_purchases
from orders
where o_date < date('2017-12-01');
```

Среднее число покупок 1,99.

Определим общее число покупателей и число тех, кто сделал более 2 покупок

```sql
select count(DISTINCT user_id)
from orders
where o_date < date('2017-12-01');
```

Всего покупателей 935 521.

```sql
select count(t.user_id)
from (
	select
		user_id,
		count(id_o) as purchases
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) t
where t.purchases > 2;
```

Часто покупающих 127 105.

Из часто покупающих выберем тех, кто делал покупки с 1 по 30 ноября 2017.

```sql
select
	t.user_id,
	t.purchases,
	t.first_purchase,
	t.last_purchase,
	t.revenue,
	t.revenue * 30 / TIMESTAMPDIFF(DAY,date(t.first_purchase),date(t.last_purchase)) as expected_purchase_per_month
from (
	select
		user_id,
		count(id_o) as purchases,
		min(o_date) as first_purchase,
		max(o_date) as last_purchase,
		sum(price) as revenue
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) t
where
	t.purchases > 2
AND
	t.last_purchase BETWEEN date('2017-11-01') AND date('2017-11-30');
```

Просуммируем ожидаемый доход за месяц по часто покупающим клиентам.

```sql
select
	sum(t.revenue * 30 / TIMESTAMPDIFF(DAY,date(t.first_purchase),date(t.last_purchase))) as total_1
from (
	select
		user_id,
		count(id_o) as purchases,
		min(o_date) as first_purchase,
		max(o_date) as last_purchase,
		sum(price) as revenue
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) t
where
	t.purchases > 2
AND
	t.last_purchase BETWEEN date('2017-11-01') AND date('2017-11-30');
```

Получили ожидаемую прибыль от первой группы 108 932 704.76

2.  Группа часто покупающих, но которые не покупали уже значительное время. Так же можем сделать вывод, из такой группы за след месяц сколько купят и на какой средний чек.

```sql
select
	sum(t.revenue * 30 / TIMESTAMPDIFF(DAY,date(t.first_purchase),date('2017-11-30'))) as total_2
from (
	select
		user_id,
		count(id_o) as purchases,
		min(o_date) as first_purchase,
		max(o_date) as last_purchase,
		sum(price) as revenue
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) t
where
	t.purchases > 2
AND
	date(t.last_purchase) < date('2017-11-01');
```

Здесь взяли средние затраты в день, начиня со дня первой покупки, заканчивая датой анализа - 30 ноября 2017.

Получили ожидаемую прибыль от второй группы 90 392 683.50 рублей.

3.  Отдельно разобрать пользователей с 1 и 2 покупками за все время.

Посчитаем, сколько времени проходит между первой и второй покупкой для покупателей, сделавших 2 и более заказов. Это будет сложный запрос, поэтому распишу его по шагам.

Сперва получаем id пользователей, у которых более одного заказа:

```sql
select user_id
from (
	select
    user_id,
    count(id_o) as orders_count
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) as t
where t.orders_count > 1;
```

Выбираем все записи из таблицы только для этих клиентов:

```sql
select t1.*
from orders t1
left join (
	select user_id
	from (
		select
			user_id,
			count(id_o) as orders_count
		from orders
		where o_date < date('2017-12-01')
		group by user_id
	) as t
	where t.orders_count > 1
) t2
on t1.user_id = t2.user_id
where t2.user_id IS NOT NULL
order by t1.user_id;
```

Выбираем первые два заказа у пользователей, сделавших более одного заказа.

```sql
select * from
(
	select
		ta.*,
		if(
			@typex=ta.user_id,
			@rownum:=@rownum+1,
			@rownum:=1+least(0,@typex:=ta.user_id)
		) rown
	from
		(
			select t1.*
			from orders t1
			left join (
				select user_id
				from (
					select
						user_id,
						count(id_o) as orders_count
					from orders
					where o_date < date('2017-12-01')
					group by user_id
				) as t
				where t.orders_count > 1
			) t2
			on t1.user_id = t2.user_id
			where t2.user_id IS NOT NULL
			order by t1.user_id
		) ta,
		(
			select @rownum:=1, @typex:='_'
		) zz
	order by user_id, o_date
) yy
where rown < 3
```

Первые 10 записей выглядят так:

| id_o    | user_id | price     | o_date     | rown |
| ------- | ------- | --------- | ---------- | ---- |
| 1241821 | 1       | 2799.300  | 2016-04-01 | 1    |
| 5212711 | 1       | 11045.300 | 2017-01-08 | 2    |
| 3281813 | 76      | 1248.100  | 2016-12-13 | 1    |
| 6125480 | 76      | 615.300   | 2017-09-11 | 2    |
| 2073453 | 90      | 1190.000  | 2016-07-16 | 1    |
| 4990364 | 90      | 544.600   | 2017-06-25 | 2    |
| 1660255 | 91      | 1073.800  | 2016-04-20 | 1    |
| 1660501 | 91      | 1397.900  | 2016-04-20 | 2    |
| 2813765 | 95      | 1099.000  | 2016-05-11 | 1    |
| 1589301 | 95      | 212.100   | 2016-08-04 | 2    |

Далее эту выборку группируем по user_id. В каждой группе будет по 2 записи - первая и вторая покупка клиента. Посчитаем средний интервал в днях между первой и второй покупкой:

```sql
select avg(days_between_first_and_second_purchase) from (
	select
		user_id,
		TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) as days_between_first_and_second_purchase
	from (
		select * from
		(
			select
				ta.*,
				if(
					@typex=ta.user_id,
					@rownum:=@rownum+1,
					@rownum:=1+least(0,@typex:=ta.user_id)
				) rown
			from
				(
					select t1.*
					from orders t1
					left join (
						select user_id
						from (
							select
								user_id,
								count(id_o) as orders_count
							from orders
							where o_date < date('2017-12-01')
							group by user_id
						) as t
						where t.orders_count > 1
					) t2
					on t1.user_id = t2.user_id
					where t2.user_id IS NOT NULL
					order by t1.user_id
				) ta,
				(
					select @rownum:=1, @typex:='_'
				) zz
			order by user_id, o_date
		) yy
		where rown < 3
	) xx
	group by user_id
) uu;
```

Среднее время составило 85.6 дней.

Аналогично рассчитаем средний интервал между второй и третьей покупкой:

```sql
select avg(days_between_second_and_third_purchase) from (
	select
		user_id,
		TIMESTAMPDIFF(DAY, min(o_date), max(o_date)) as days_between_second_and_third_purchase
	from (
		select * from
		(
			select
				ta.*,
				if(
					@typex=ta.user_id,
					@rownum:=@rownum+1,
					@rownum:=1+least(0,@typex:=ta.user_id)
				) rown
			from
				(
					select t1.*
					from orders t1
					left join (
						select user_id
						from (
							select
								user_id,
								count(id_o) as orders_count
							from orders
							where o_date < date('2017-12-01')
							group by user_id
						) as t
						where t.orders_count > 2
					) t2
					on t1.user_id = t2.user_id
					where t2.user_id IS NOT NULL
					order by t1.user_id
				) ta,
				(select @rownum:=1, @typex:='_') zz
			order by user_id, o_date
		) yy
		where rown BETWEEN 2 AND 3
	) xx
	group by user_id
) uu;
```

Среднее время составило 70.3 дня.

Далее ещё немного подготовительных расчётов.

Найдём вероятности того что клиент, сделав один заказ, сделает и второй. И того что клиент, сделав два заказа, сделает третий.

```sql
select count(user_id), t.orders_count
from (
	select
		user_id,
		count(id_o) as orders_count
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) as t
group by t.orders_count;
```

Первые 10 строк результата:

| count(user_id) | orders_count |
| -------------- | ------------ |
| 710608         | 1            |
| 97808          | 2            |
| 40177          | 3            |
| 22527          | 4            |
| 14526          | 5            |
| 9944           | 6            |
| 7313           | 7            |
| 5524           | 8            |
| 4293           | 9            |
| 3413           | 10           |

Общее число покупателей 935 521.

Покупателей, сделавших только одну покупку 710 608.

Покупателей, сделавших только две покупки 97 808.

Из этих данных найдём вероятность того что пользователь с одним заказом сделает второй заказ:

`second_purchase_probability = 1 - 710608/935521 = 0.24`

Вероятность того, что пользователь с двумя заказами сделает третий заказ:

`third_purchase_probability = 1 - 97808 / (935521 - 710608) = 0.57`

Разберём пользователей, совершивших только один заказ и у которых потенциальная дата второго заказа выпадет на декабрь 2017 года.

```sql
select sum(revenue)
from (
	select
		user_id,
		count(id_o) as orders_count,
		max(o_date) as purchase_date,
		sum(price) as revenue
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) as t
where t.orders_count = 1
and TIMESTAMPDIFF(day, purchase_date, date('2017-12-31')) < 86;
```

Получили сумму 234 502 888.2 рублей. Пока примем что второй заказ будет на ту же сумму и 24% клиентов сделают этот второй заказ.

Тогда рассчетная прибыль от этой группы клиентов в декабре 2017 составит 56 374 494.32 рублей.

Аналогичный расчёт для клиентов, сделавших два заказа.

```sql
select sum(revenue)
from (
	select
		user_id,
		count(id_o) as orders_count,
		max(o_date) as purchase_date,
		sum(price) as revenue
	from orders
	where o_date < date('2017-12-01')
	group by user_id
) as t
where t.orders_count = 2
and TIMESTAMPDIFF(day, purchase_date, date('2017-12-31')) < 71;
```

Получили сумму 81 597 610.5 рублей. Примем что третий заказ будет на ту же сумму и 57% клиентов сделают этот третий заказ.

Рассчетная прибыль от этой группы клиентов в декабре 2017 составит 46 113 227.26 рублей.

4.  В итоге у вас будет прогноз ТО и вы сможете его сравнить с фактом и оценить грубо разлет по данным.

```sql
select sum(price) as revenue
from orders
where o_date >= date('2017-12-01')
```

| revenue       |
| ------------- |
| 322948401.300 |

Фактически в декабре 2017 было сдлано покупок на 322 948 401.3 рублей.

Расчётная сумма

```text
 108 932 704.76 +
  90 392 683.50 +
  56 374 494.32 +
  46 113 227.26 =
 301 813 109.84 рубля.
```

Погрешность составила 6,5%.

</details>

<details>
	<summary>Урок 6. Хранилища данных для анализа. Логика работы рекомендательного сервиса.</summary>

1. Продумать логику работы рекомендательного сервиса для сайта дестких товаров

   Сперва определяем, есть ли у нас достаточная информация на клиента, чтобы ему что-то рекомендовать.

   Если нет, то всё просто - ждём когда он что-то положит в корзину, или проведёт секунд 15 в карточке одного товара. Формируем рекомендации, опираясь на этот товар. Тут поджойдёт всё: сопутствующие товары, товары конкуренты. Приоритетно показать из подобранного списка товары-маркеры и проследить, заинтересуют ли они клиента. Это даст больше информации о клиенте.

   Если данные на клиента есть, то он уже отнесён к какой-то категории. Предлагаем самые популярные товары среди клиентов этой же категории. Исключаем из показа товары-конкуренты, так как клиент уже опытный и скорее всего знвает что хочет. Товары-конкуренты, даже более дешёвые его скорее всего не заинтересуют. Остаются сопутствующие товары. В сформированном списке можно не делать акцент на товарах-маркерах, так как мы не гонимся за новой информацией о клиенте.

2. На каких страницах может быть показан блок рекомендаций

   Думаю, сразу показывать посетителю рекомендации не стоит, так как можно замусорить информацию на экране.

   Можно показывать рекомендации уже в корзине. Когда клент сделал выбор, можно предложить ему сопутствующий или альтернативный более дешёвый товар с большей маржинальностью (если вообще такое встречается).

   Так же часто бывает что товары в корзине видны где-то сбоку, в процессе покупки. В этой же секции так же можно показывать рекомендации. Если есть хорошая мета на клиента, можно показывать рекомендательный блок в этой секции не дожидаясь что клиент что-то положит в корзину.

   В карточке товара.

   Основной посыл в том, что клиент уже что-то положил в корзину или провёл какое-то время в карточке товара и можно точнее определить его категорию и более удачно сделать предложение.

3. Какие вообще вы видите типы блоков рекомендаций

   - "С этим часто покупают..."
   - "Аналогичный товар дешевле" (но с большей маржой) - этот кейс вызывает довериие у клиентов.

   Вообще, чем больше информации о товарах, сравнительных графиков, более выгодных для климента предложений, тем лучше. Эта кажущаяся забота о клиенте всегда вызывает доверие. Может быть актуально для постоянных покупателей. Минус к прибыли, плюс к репутации фирмы.

4. Логика подбора товаров

   Тут нужно расставлять приоритеты.

   Если клиент новый, то лучше подбирать сопутствующие товары, из них приоритетнее товары-маркеры.

   Если клиент старый, то так же сопутствующие товары, без приоритета на товары-маркеры.

   Если клиент сделал 2-3 покупки, то равнозначно предлагаем и сопутствующие товары, и более дешевые товары-конкуренты (зарабатываем репутацию у клиента). В этом случае товаров-маркеры никак невыделяем по приоритету.

5. Продумать написание скрипта, который в типовой структуре БД инет-магазина найдет наиболее часто встречаемый товар с добавленным в корзину товаром.

```sql
select product_id, sum(quantity) as `match` from basket
where order_id IN (
	select order_id from basket
	where product_id = 196816
)
group by product_id
order by `match` DESC
limit 5;
```

| product_id | match  |
| ---------- | ------ |
| 196816     | 458.00 |
| 177628     | 198.00 |
| 177873     | 165.00 |
| 177539     | 151.00 |
| 177744     | 142.00 |

Как и ожидалось, самый часто встречаемый товар тот, по которому искали совпадение.

Тот же запрос для товара с id 196817:

| product_id | match  |
| ---------- | ------ |
| 196817     | 516.00 |
| 196889     | 26.00  |
| 177524     | 23.00  |
| 177415     | 20.00  |
| 203019     | 20.00  |

</details>

<details>
	<summary>Урок 7. Системы визуализации данных. Работа с группами товаров.</summary>

В заказах часто одновлеменнно встречаются товар1 и товар2. Надо найти на каждую такую существующую группу товаров общий ТО, кот. они делают, сколько делает в этих группах каждый из товаров и еще найти сколько каждый из этих товаров делает не в этих группа.
К примеру: Товар1 и Товар2. Группа Т1 и Т2 вместе (когда встречается в заказах одновремнно) генерит столько-то. Товар Т1, когда в заказах идет вместе с Т2, генерит столько-то. Аналогично для Т2. Т1 в заказах, где нет Т2, генерит столько-то. Т2 в заказах, где нет Т1, генерит столько-то.

Подобным образом надо разобрать все группы. И в итоге мы будем понимать как ведет себя товар один как таковой и как во всех возможных (в которых он стречался) группах. Подумайте, как бы вы подошли к решению такой задачи.

Исходник баскета: https://drive.google.com/open?id=1Mny5vMvBMCanejc9AEA8y4SVwvgWGd0S

Запрос будет сложным, поэтому распишу его по шагам. В таблице имеются следующие колонки: `id`, `order_id`, `product_id`, `quantity`, `price`.

Сперва посчитаем стоимости каждого заказа, сгруппировав данные по `order_id`.

```sql
SELECT
	order_id,
	sum(quantity * price) as order_revenue
FROM basket
GROUP BY order_id;
```

| order_id | order_revenue |
| -------- | ------------- |
| 129      | 1310.0000     |
| 130      | 260.0000      |
| 131      | 260.0000      |
| 132      | 1120.0000     |
| 133      | 3230.0000     |
| 134      | 1380.0000     |
| 135      | 9500.0000     |
| 137      | 1800.0000     |
| 138      | 22350.0000    |
| 141      | 820.0000      |

Далее сформируем пары товаров, используя join таблицы самой на себя. Так как одна пара товаров в одном заказе может встретиться только один раз, то сразу же к этому запросу подошьём стоимости заказов, вычисленные на предыдущем шаге:

```sql
SELECT
	concat(b1.product_id, '_', b2.product_id) as product_pair,
	b1.product_id as p1,
	b2.product_id as p2,
	b1.order_id as order_id,
	b1.quantity * b1.price as p1_at_group,
	b2.quantity * b2.price as p2_at_group,
	orders.order_revenue as order_revenue
FROM basket b1
JOIN basket b2
ON b1.order_id = b2.order_id
JOIN (
	SELECT
		order_id,
		sum(quantity * price) as order_revenue
	FROM basket
	GROUP BY order_id
) orders
ON orders.order_id = b1.order_id
WHERE b1.product_id < b2.product_id;
```

| product_pair  | p1     | p2     | order_id | p1_at_group | p2_at_group | order_revenue |
| ------------- | ------ | ------ | -------- | ----------- | ----------- | ------------- |
| 109614_118060 | 109614 | 118060 | 135      | 4760.0000   | 410.0000    | 9500.0000     |
| 109614_113544 | 109614 | 113544 | 135      | 4760.0000   | 4330.0000   | 9500.0000     |
| 113544_118060 | 113544 | 118060 | 135      | 4330.0000   | 410.0000    | 9500.0000     |
| 115865_134873 | 115865 | 134873 | 137      | 710.0000    | 550.0000    | 1800.0000     |
| 115865_136088 | 115865 | 136088 | 137      | 710.0000    | 540.0000    | 1800.0000     |
| 134873_136088 | 134873 | 136088 | 137      | 550.0000    | 540.0000    | 1800.0000     |
| 99131_130047  | 99131  | 130047 | 138      | 1120.0000   | 21230.0000  | 22350.0000    |
| 114887_124370 | 114887 | 124370 | 144      | 340.0000    | 270.0000    | 2080.0000     |
| 114887_120212 | 114887 | 120212 | 144      | 340.0000    | 420.0000    | 2080.0000     |
| 114248_124370 | 114248 | 124370 | 144      | 380.0000    | 270.0000    | 2080.0000     |

Здесь `p1_at_group` и `p2_at_group` - вклад каждого товара в группе, а `order_revenue` - стоимость всего заказа для данной группы.

Так же посчитаем суммарный товарооборот для каждого товара, сгруппировав данные из таблицы по `product_id`:

```sql
SELECT
	product_id,
	sum(quantity * price) as total_revenue
FROM basket
GROUP BY product_id;
```

| product_id | total_revenue |
| ---------- | ------------- |
| 94602      | 1284.0000     |
| 94605      | 171650.0000   |
| 94606      | 1280.0000     |
| 94607      | 720.0000      |
| 94608      | 7820.0000     |
| 94616      | 2940.0000     |
| 94619      | 520.0000      |
| 94626      | 300.0000      |
| 94627      | 4070.0000     |
| 94628      | 1800.0000     |

Эти данные будем склеивать с данными по группе для товаров 1 и 2, чтобы иметь представление, какая доля от общего товарооборота по каждому товару приходится на конкретную группу.

Результирующий запрос:

```sql
SELECT
	gr.product_pair,
	gr.p1,
	gr.p2,
	gr.orders_count,
	gr.p1_groups_revenue as p1_at_group,
	gr.p2_groups_revenue as p2_at_group,
	gr.orders_revenue,
	single_p1.total_revenue as p1_total,
	single_p2.total_revenue as p2_total
FROM (
	SELECT
		pairs.product_pair as product_pair,
		SUBSTRING_INDEX(SUBSTRING_INDEX(pairs.product_pair, '_', 1), '_', -1) as p1,
		SUBSTRING_INDEX(SUBSTRING_INDEX(pairs.product_pair, '_', 2), '_', -1) as p2,
		count(pairs.order_id) as orders_count,
		sum(pairs.p1_group_revenue) as p1_groups_revenue,
		sum(pairs.p2_group_revenue) as p2_groups_revenue,
		sum(pairs.order_revenue) as orders_revenue
	FROM (
		SELECT
			concat(b1.product_id, '_', b2.product_id) as product_pair,
			b1.product_id as p1,
			b1.order_id as order_id,
			b1.quantity * b1.price as p1_group_revenue,
			b2.quantity * b2.price as p2_group_revenue,
			orders.order_revenue as order_revenue
		FROM basket b1
		JOIN basket b2
		ON b1.order_id = b2.order_id
		JOIN (
			SELECT
				order_id,
				sum(quantity * price) as order_revenue
			FROM basket
			GROUP BY order_id
		) orders
		ON orders.order_id = b1.order_id
		WHERE b1.product_id < b2.product_id
	) pairs
	GROUP BY pairs.product_pair
) gr
LEFT JOIN (
	SELECT
		product_id,
		sum(quantity * price) as total_revenue
	FROM basket
	GROUP BY product_id
) single_p1
ON gr.p1 = single_p1.product_id
LEFT JOIN (
	SELECT
		product_id,
		sum(quantity * price) as total_revenue
	FROM basket
	GROUP BY product_id
) single_p2
ON gr.p2 = single_p2.product_id;
```

| product_pair  | p1     | p2     | orders_count | p1_at_group | p2_at_group | orders_revenue | p1_total | p2_total  |
| ------------- | ------ | ------ | ------------ | ----------- | ----------- | -------------- | -------- | --------- |
| 100000_100990 | 100000 | 100990 | 1            | 420.0000    | 110.0000    | 2110.0000      | 840.0000 | 1070.0000 |
| 100000_104695 | 100000 | 104695 | 1            | 420.0000    | 250.0000    | 4530.0000      | 840.0000 | 1450.0000 |
| 100000_110872 | 100000 | 110872 | 1            | 420.0000    | 110.0000    | 2110.0000      | 840.0000 | 490.0000  |
| 100000_121369 | 100000 | 121369 | 1            | 420.0000    | 180.0000    | 4530.0000      | 840.0000 | 1910.0000 |
| 100000_123980 | 100000 | 123980 | 1            | 420.0000    | 190.0000    | 4530.0000      | 840.0000 | 6750.0000 |
| 100000_124626 | 100000 | 124626 | 1            | 420.0000    | 210.0000    | 4530.0000      | 840.0000 | 210.0000  |
| 100000_127348 | 100000 | 127348 | 1            | 420.0000    | 190.0000    | 4530.0000      | 840.0000 | 5700.0000 |
| 100000_127391 | 100000 | 127391 | 1            | 420.0000    | 120.0000    | 2110.0000      | 840.0000 | 800.0000  |
| 100000_127617 | 100000 | 127617 | 1            | 420.0000    | 690.0000    | 2110.0000      | 840.0000 | 7590.0000 |
| 100000_132584 | 100000 | 132584 | 1            | 420.0000    | 200.0000    | 2110.0000      | 840.0000 | 8992.0000 |

Здесь дважды использовали left join результата из предыдущего запроса, так как нам нужны данные по товарообороту и для p1 и для p2, а эти товары в группах не повторяются (то есть если есть группа p1p2, то группа p2p1 уже не встретится). В результате видим вклад каждого товара в группу, ценность всей группы по всем заказам и отдельный вклад каждого товара.

Предположу, как можно использовать получившийся результат.

1. Можно отсортировать результат по `gr.orders_count`, чтобы понять какие товары чаще всего покупают вместе. Например, товары 193396 и 193463 вместе покупали чаще всего - 173 раза. Товары 193574 и 193583 - 138 раз.

2. Отношение `p1_at_group` к `p1_total` показывает, покупают ли товар чаще в паре с другими товарами, или чаще его покупают отдельно.

</details>

<details>
	<summary>Урок 8. Типовые структуры БД. Изменение групп пользователей при RFM анализе.</summary>

ДЗ делаем по бд orders
В качестве ДЗ сделаем карту поведения пользователей. Мы обсуждали, что всех пользователей можно разделить, к примеру, на New (совершили только 1 покупку), Regular (совершили 2 или более на сумму не более стольки-то), Vip (совершили дорогие покупки и достаточно часто), Lost (раньше покупали хотя бы раз и с даты последней покупки прошло больше 3 месяцев). Вся база должна войти в эти гурппы (т.е. каждый пользователь должен попадать только в одну из этих групп).

Задача:

1. Уточнить критерии групп New,Regular,Vip,Lost

Lost - с момента последней покупки прошло более 60 дней;
New - с момента первой покупки прошло менее 60 дней (включительно) и была только одна покупка;
Regular - сумма покупок менее 5000, было более 1 покупки и клиент не попал в группу Lost;
Vip - сумма покупок более 5000 (включительно), было более 1 покупки и клиент не попал в группу Lost.

2. По состоянию на 1.01.2017 понимаем, кто попадает в какую группу, подсчитываем кол-во пользователей в каждой.

Используем переменную для задания даты анализа

```sql
set @analys_date = date('2017-01-01');
```

В основном запросе воспользуемся тем что конструкция `case when then` возвращает первое успешное совпадение. Сперва отсеим потерянных климентов, из оставшихся выберем новых (с одной покупкой), из оставшихся выберем регуляров (с суммарной покупкой менее 5000р), остальные попадут в Vip.

```sql
select
	category,
	count(user_id) as users_count,
	sum(total_revenue) as total_revenue,
	sum(orders_count) as orders_count
from (
	select
		user_id,
		count(id_o) as orders_count,
		sum(price) as total_revenue,
		case
			when TIMESTAMPDIFF(DAY, max(o_date), @analys_date) > 60 then "Lost"
			when count(id_o) = 1 then "New"
			when sum(price) < 5000 then "Regular"
			else "Vip"
			end as category
	from orders
	where o_date < @analys_date
	group by user_id
) group_by_user_id
group by category;
```

| category | users_count | total_revenue | orders_count |
| -------- | ----------- | ------------- | ------------ |
| Lost     | 291973      | 902733049.400 | 414086       |
| New      | 98243       | 205359002.800 | 98243        |
| Regular  | 22151       | 62450202.500  | 62752        |
| Vip      | 32725       | 634480397.950 | 286265       |

Для проверки, посчитаем число пользователей в выборке

```sql
select count(distinct user_id) from orders where o_date < @analys_date;
```

| count(distinct user_id) |
| ----------------------- |
| 445092                  |

Число пользователей сходится, значит никого не потеряли.

3. По состоянию на 1.02.2017 понимаем, кто вышел из каждой из групп, а кто вошел.

```sql
set @analys_date = date('2017-02-01');
```

| category | users_count | total_revenue  | orders_count |
| -------- | ----------- | -------------- | ------------ |
| Lost     | 347612      | 1112856133.200 | 515021       |
| New      | 91484       | 200968670.700  | 91484        |
| Regular  | 19807       | 56078969.100   | 54900        |
| Vip      | 30708       | 631172272.850  | 283628       |

4. Аналогично смотрим состояние на 1.03.2017, понимаем кто вышел из каждой из групп, а кто вошел.

```sql
set @analys_date = date('2017-03-01');
```

| category | users_count | total_revenue  | orders_count |
| -------- | ----------- | -------------- | ------------ |
| Lost     | 411553      | 1371155888.900 | 645737       |
| New      | 74336       | 183864223.900  | 74336        |
| Regular  | 15316       | 43815328.200   | 41719        |
| Vip      | 26571       | 573547544.150  | 255316       |

5. В итоге делаем вывод, какая группа уменьшается, какая увеличивается и продумываем, в чем может быть причина.

Отобразим динамику на графиках:

![Users count](/images/pic_1_users_count.png)

![Total revenue](/images/pic_2_total_revenue.png)

![Orders count](/images/pic_3_orders_count.png)

Видно что число потерянных пользователей растёт. Это результат естественного вымирания пользовательской аудитории.

Так же виден сильный спад числа новых, регулярных и вип пользователей на дату анализа 1 марта 2017. Это может быть связано с тем что для этих групп временной диапазон даты последней покупки ограничен двумя месяцами до даты анализа. То есть многие пользователи сделали покупки в декабре и перестали их делать в январе. Если бы мы считали потерянными клиентов, сделавших последнюю покупку более месяца назад, то этот спад пришелся бы на 1 февраля 2017.

</details>

[Курсовой проект](https://github.com/dimireme/db_for_data_scince/tree/master/coursework)
