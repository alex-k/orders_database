#Задача 1

Для решения данной задачи мне бы потребовалось провести определенную нормализацию данных, а именно выгрузить во вспомогательные таблицы сгруппированные по определенным критериям данные. 

Так как в части заданий требуемый результат имеет статистический характер (самый популярный период, области с самой высокой плотностью точек), а количество данных относительно велико, то достаточно выгрузить лишь часть данных, достаточную для получения достоверного результата. 

Я полагаю что для задачи о популярных адресах будет достаточно 1 млн. Заказов, для определения популярности времен года 1 тысячи точек на каждый год. 
Также я считаю разумным разделять хранение и анализ данных, таким образом минимизировав влияние новой функциональности по анализу на работу исходного приложения, осуществляющего работу с сырыми данными.

Для экспорта данных возможно использовать batch процедуру для изначального экспорта и триггеры на уровне БД или приложения для изменяемых впоследствии данных. 
Пример запроса для получения случайной записи из таблицы:

```
SELECT *
  FROM raw_records JOIN
       (SELECT CEIL(RAND() *
                     (SELECT MAX(id)
                        FROM raw_records)) AS sid)
        AS r2
 WHERE raw_orders >= r2.sid
 LIMIT 1;
```


Скорость выгрузки для MySQL составляет 10-25 тыс. записей/сек на сервере средней конфигурации ( AWS EC2 m2.4xl, 8 cpu cores, 64Gb ram), что позволяет прогнозировать экспорт 1млн. исходных записей за 40-100 секунд и всего набора данных за 1-3 часа.


##Вопрос 1 (области содержащие популярные адреса): 

таблица order_address

```
CREATE TABLE `orders` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `order_number` varchar(255) DEFAULT NULL,
  `order_date` timestamp NULL DEFAULT NULL,
  `shipping_address` varchar(255) DEFAULT NULL,
  `shipping_coordinates` varchar(255) DEFAULT NULL,
  `quadtree_coordinates` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `order_number` (`order_number`),
  KEY `quadtree_coordinates` (`quadtree_coordinates`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

Реализация зависит от используемой БД, для некоторых доступна функциональность оптимизированного хранения географических/геометрических данных и построения двумерных индексов, такие как OpenGIS (https://dev.mysql.com/doc/refman/5.5/en/opengis-geometry-model.html).  

Если же рассматривать общий случай то можно на первом этапе ограничиться построением QadTree индекса. В простейшем случае карта разбивается мысленно на квадранты, каждому квадранту присваивается индекс 0-3, далее каждый квадрант делится на 4 подквадранта с аналогичными индексами и так далее. 

Каждая точка находится в квадранте наименьшего размера, и можно составить ее индекс вплоть до квадранта верхнего уровня (как пример, справа налево 01031121). Для дерева из 32х уровней и обхватом карты во всю планету мы получим размер наименьшего квадранта порядка 1 см, что достаточно для нашей задачи. Индекс можно сжать в строку или int для оптимизации, но я остановлюсь на наивном представлении.

Для того чтобы определить количество точек в квадранте достаточно сделать запрос вида

```
 select count(*) from orders where quadtree_coordinates like '123%';
```

Таким образом мы можем рекурсивно обойти дерево, откидывая квадранты содержащие менее чем 10% точек и найти области удовлетворяющие нашему критерию. Для более точного анализа можно найти соседние квадранты малой площади, в сумме содержащие 10% точек, а также использовать библиотеки работы с геометрическими данными для нахождения окружностей.

Для подсчета общего кол-ва записей можно использовать разные способы, зависит от BD, в MySQL можно использовать метаинформацию по таблице

```
 show table status like 'order_address';
```

Для определения самых популярных товаров из найденной области достаточно сделать join с таблицей products, которую также можно выгрузить из исходных данных.

##Вопрос 2 (времена года):

Таблица
```

CREATE TABLE `season_orders` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `order_number` varchar(255) DEFAULT NULL,
  `order_date` timestamp NULL DEFAULT NULL,
  `year_and_season` varchar(32) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `order_number` (`order_number`),
  KEY `year_and_season` (`year_and_season`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

Запрос
```
 select `year_and_season`, count(id) as orders from season_order group by year_and_season order by year_and_season;
```


## Вопрос 3 (адреса пользователей)


Для решения данной задачи можно сделать экспорт в 2 таблицы, одну содержащую все адреса доставки каждого пользователя, и вторую, содержащую агрегированные данные по количеству адресов.

```

CREATE TABLE `user_shipping_addresses` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `customer_name` varchar(255) NOT NULL DEFAULT '',
  `shipping_address` varchar(255) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  UNIQUE KEY `customer_name` (`customer_name`,`shipping_address`)
) ENGINE=InnoDB AUTO_INCREMENT=131071 DEFAULT CHARSET=utf8;
```


```
CREATE TABLE `user_shipping_address_count` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `customer_name` varchar(255) NOT NULL DEFAULT '',
  `address_count` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `customer_name` (`customer_name`)
) ENGINE=InnoDB AUTO_INCREMENT=32773 DEFAULT CHARSET=utf8;
```


```
replace `user_shipping_addresses` (customer_name,shipping_address ) select `customer_name`, `shipping_address` from `raw_records`;
```

```
INSERT INTO `user_shipping_address_count` (`customer_name`, `address_count`)
select customer_name, 1 from user_shipping_addresses on duplicate key update address_count = address_count +1;
```

Для выбора пользователей:

```
select * from user_shipping_address_count where address_count = 4;
```
