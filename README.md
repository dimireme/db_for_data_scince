# db_for_data_scince

Home work at GB course "Databases for datascince"

### Lesson 1.

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

5) Найти кол-во пользователей, кот покупали в одном году и перестали покупать в следующем.

   ```sql
   select YEAR(o_date) as 'year', count(DISTINCT user_id) from orders_short group by YEAR(o_date);
   ```

   **не доделано**

6. Найти ID самого активного по кол-ву покупок пользователя.

   ```sql
   select user_id, count(id_o) as orders from orders_short group by user_id order by orders DESC LIMIT 1;
   ```

   | user_id | orders |
   | ------- | ------ |
   | 765861  | 3183   |
