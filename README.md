# db_for_data_scince

Домашние работы по курсу "Базы данных для аналитиков" от GeekBrains.

<details>
  <summary>Урок 1. Аналитика в бизнес-задачах.</summary>

1. Залить в свою БД данные по продажам (часть таблицы Orders в csv, исходник [здесь](https://drive.google.com/drive/folders/1C3HqIJcABblKM2tz8vPGiXTFT7MisrML?usp=sharing).

2. Проанализировать, какой период данных выгружен

   ```sql
   select min(o_date), max(o_date) from orders_short;
   ```

   | min(o_date) | max(o_date) |
   | ----------- | ----------- |
   | 2001-01-20  | 2031-12-20  |

3) Посчитать кол-во строк, кол-во заказов и кол-во уникальных пользователей, кот совершали заказы.

   ```sql
   select count(id_o) as total, count(DISTINCT id_o) as unique_orders, count(DISTINCT user_id) as unique_users from orders_short;
   ```

   | total   | unique_orders | unique_users |
   | ------- | ------------- | ------------ |
   | 2002804 | 2002804       | 1015119      |

4. По годам посчитать средний чек, среднее кол-во заказов на пользователя, сделать вывод , как изменялись это показатели Год от года.

   ```sql
   select YEAR(o_date) as 'year', ROUND(AVG(price), 0) as 'avg_price', count(id_o) / count(DISTINCT user_id) as avg_orders from orders_short group by YEAR(o_date);
   ```

    <details>
      <summary>Результат</summary>

   | year | avg_price | avg_orders |
   | ---- | --------- | ---------- |
   | 2001 | 2329709   | 1.1825     |
   | 2002 | 2356190   | 1.1801     |
   | 2003 | 2352382   | 1.1852     |
   | 2004 | 2355992   | 1.1785     |
   | 2005 | 2314750   | 1.1866     |
   | 2006 | 2275903   | 1.1985     |
   | 2007 | 2291585   | 1.1897     |
   | 2008 | 2279671   | 1.1962     |
   | 2009 | 2253987   | 1.1864     |
   | 2010 | 2258623   | 1.1873     |
   | 2011 | 2261050   | 1.2001     |
   | 2012 | 2265938   | 1.1924     |
   | 2013 | 2289328   | 1.2036     |
   | 2014 | 2287217   | 1.1913     |
   | 2015 | 2144858   | 1.2259     |
   | 2016 | 2193478   | 1.1839     |
   | 2017 | 2239746   | 1.2146     |
   | 2018 | 2249528   | 1.2286     |
   | 2019 | 2144138   | 1.2215     |
   | 2020 | 2159371   | 1.2125     |
   | 2021 | 2216198   | 1.1990     |
   | 2022 | 2262312   | 1.1979     |
   | 2023 | 2282133   | 1.2000     |
   | 2024 | 2265576   | 1.2050     |
   | 2025 | 2274696   | 1.1853     |
   | 2026 | 2272565   | 1.1718     |
   | 2027 | 2332798   | 1.1764     |
   | 2028 | 2294473   | 1.1809     |
   | 2029 | 2352479   | 1.1751     |
   | 2030 | 2347706   | 1.1737     |
   | 2031 | 2279842   | 1.1523     |

    </details>

    <details>
      <summary>Графики</summary>

   Зависимость средней цены заказа от года

   ![Зависимость средней цены заказа от года](https://i.postimg.cc/c41fhXcn/graph1.png)

   Зависимость среднего количества заказов на пользователдя от года

   ![Зависимость среднего количества заказов на пользователдя от года](https://i.postimg.cc/CMsjbWb0/graph2.png)

    </summary>

5) Найти кол-во пользователей, кот покупали в одном году и перестали покупать в следующем.

   ```sql
    select count(t16.user_id) as 'count' from
      (select DISTINCT user_id from orders_short where YEAR(o_date) = 2016) t16
    left join
      (select DISTINCT user_id from orders_short where YEAR(o_date) = 2017) t17
    on t16.user_id = t17.user_id
    where t17.user_id is null;
   ```

   | count |
   | ----- |
   | 50338 |

6. Найти ID самого активного по кол-ву покупок пользователя.

   ```sql
   select user_id, count(id_o) as orders from orders_short group by user_id order by orders DESC LIMIT 1;
   ```

   | user_id | orders |
   | ------- | ------ |
   | 765861  | 3183   |

</details>

<details>
  <summary>Урок 3. Типовые методы анализа данных. RFM-анализ</summary>

Главная задача: сделать RFM-анализ на основе данных по продажам за 2 года.

1. Определяем критерии для каждой буквы R, F, M (т.е. к примеру, R – 3 для клиентов, которые покупали <= 30 дней от последней даты в базе, R – 2 для клиентов, которые покупали > 30 и менее 60 дней от последней даты в базе и т.д.)

| номер | r               | f                | m                   |
| ----- | --------------- | ---------------- | ------------------- |
| 1     | 60 < days       | 20 <= period     | spend < 1000        |
| 2     | 30 < days <= 60 | 10 <= period <20 | 1000 <= spend <5000 |
| 3     | days <= 30      | period < 10      | 5000 <= spend       |

При этом если пользователь совершил менее 4-х покупок, при определении периода f, он попадёт в категорию 1.

2. Для каждого пользователя получаем набор из 3 цифр (от 111 до 333, где 333 – самые классные пользователи)

```sql
DROP TABLE IF EXISTS `rfm_analys`;
CREATE TABLE `rfm_analys`
SELECT
	user_id,
	min(o_date) as first_activity,
	max(o_date) as last_activity,
	count(id_o) as orders_count,
	sum(price) as total_price,
	CASE
		WHEN count(id_o) < 4 THEN "1"
		ELSE (
			CASE
				WHEN (TIMESTAMPDIFF(DAY,min(o_date),max(o_date)) / (count(id_o) - 1)) < 10 THEN "3"
				WHEN (TIMESTAMPDIFF(DAY,min(o_date),max(o_date)) / (count(id_o) - 1)) >= 10 AND (TIMESTAMPDIFF(DAY,min(o_date),max(o_date)) / (count(id_o) - 1)) < 20 THEN "2"
				ELSE "1" END
		) END
	as f,
	CASE
		WHEN sum(price) < 1000 THEN "1"
		WHEN sum(price) >= 1000 AND sum(price) < 5000 THEN "2"
		ELSE "3" end  AS m,
	CASE
		WHEN TIMESTAMPDIFF(DAY,max(o_date),date('2018-01-01')) >= 0 AND TIMESTAMPDIFF(DAY,max(o_date),date('2018-01-01')) < 30 THEN "1"
       	WHEN TIMESTAMPDIFF(DAY,max(o_date),date('2018-01-01')) >= 30 AND TIMESTAMPDIFF(DAY,max(o_date),date('2018-01-01')) < 60 THEN "2"
  		ELSE "3" end  AS r
FROM orders
WHERE YEAR(o_date) >= 2016 AND YEAR(o_date) <= 2017
GROUP BY user_id;
```

3. Вводим группировку, к примеру, 333 и 233 – это Vip, 1XX – это Lost, остальные Regular ( можете ввести боле глубокую сегментацию)

```sql
SELECT count(user_id) as 'count_users', sum(total_price) as sum_price, r, f, m FROM rfm_analys GROUP BY r, f, m;
```

<details>
  <summary>результат</summary>

| count_users | sum_price    | r   | f   | m   |
| ----------- | ------------ | --- | --- | --- |
| 2278        | 1415626.80   | 1   | 1   | 1   |
| 4864        | 11119192.00  | 1   | 1   | 2   |
| 909         | 9258659.90   | 1   | 1   | 3   |
| 1           | 2183.30      | 1   | 2   | 2   |
| 12          | 1209730.90   | 1   | 2   | 3   |
| 4           | 9448.60      | 1   | 3   | 2   |
| 17          | 4590434.10   | 1   | 3   | 3   |
| 1728        | 1062237.40   | 2   | 1   | 1   |
| 3485        | 8147398.00   | 2   | 1   | 2   |
| 1072        | 12651483.60  | 2   | 1   | 3   |
| 6           | 882138.60    | 2   | 2   | 3   |
| 3           | 11225.90     | 2   | 3   | 2   |
| 13          | 2347355.50   | 2   | 3   | 3   |
| 29864       | 17187681.70  | 3   | 1   | 1   |
| 51251       | 115306470.30 | 3   | 1   | 2   |
| 10586       | 109157612.90 | 3   | 1   | 3   |
| 9           | 28374.50     | 3   | 2   | 2   |
| 22          | 1296271.90   | 3   | 2   | 3   |
| 5           | 3701.60      | 3   | 3   | 1   |
| 71          | 206940.30    | 3   | 3   | 2   |
| 167         | 4687843.30   | 3   | 3   | 3   |

</details>

Всего пользователей потраченных ими денег:

```sql
SELECT count(user_id), sum(total_price) FROM rfm_analys;
```

| count(user_id) | sum(total_price) |
| -------------- | ---------------- |
| 106367         | 300582011.10     |

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

4. Для каждой группы из п. 3 находим кол-во пользователей, кот. попали в них и % товарооборота, которое они сделали на эти 2 года.

```sql
SELECT
	sum(total_price) as total_spend,
	concat(round(( sum(total_price)/ (SELECT sum(total_price) FROM rfm_analys) * 100 ),2),'%') AS percentage,
	count(user_id) as users_count,
	category
FROM rfm_analys
GROUP BY category
ORDER BY total_spend DESC;
```

| total_spend  | percentage | users_count | category |
| ------------ | ---------- | ----------- | -------- |
| 265941536.70 | 88.48%     | 98102       | regular  |
| 27605275.60  | 9.18%      | 8085        | lost     |
| 7035198.80   | 2.34%      | 180         | vip      |

5. Проверяем, что общее кол-во пользователей бьется с суммой кол-ва пользователей по группам из п. 3 (если у вас есть логические ошибки в создании групп, у вас не собьются цифры). То же самое делаем и по деньгам.

Количество пользователей в пункте 4 `98102 + 8085 + 180 = 106367` совпадает с количеством пользователей в пункте 3.

Количество потраченных денег в пункте 4 `265941536.70 + 27605275.60 + 7035198.80 = 300582011.1` совпадает со значением в пункте 3.

   </details>
