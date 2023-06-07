<pre>
Запрос (MySQL):

WITH unpopular_reservations AS (
    SELECT room_number, person_id, check_in_date
    FROM reservations
	WHERE room_number NOT IN (
	  SELECT room_number
	  FROM reservations
	  WHERE reserved_at >= (NOW() - INTERVAL 1 YEAR)
	  GROUP BY room_number
	  HAVING count(*) >= 2
	)
)
SELECT unpopular_reservations.room_number, persons.first_name, persons.last_name
FROM unpopular_reservations
INNER JOIN (
	SELECT room_number, MAX(check_in_date) as max_date
	FROM unpopular_reservations
	WHERE check_in_date <= CURDATE()
	GROUP BY room_number
) max_dates ON max_dates.room_number = unpopular_reservations.room_number AND max_dates.max_date = unpopular_reservations.check_in_date
INNER JOIN persons ON persons.id = unpopular_reservations.person_id;


Алгоритм:
1). Достаю все комнаты, имеющие менее двух бронирований в прошлом году (NOT IN reserved_at >= ...)
2). Достаю все брони по этим "непопулярным" комнатам (unpopular_reservations)
3). Нахожу последние брони по каждой комнате (INNER JOIN max_date)
4). Цепляю имена юзеров (INNER JOIN persons)


Тестирование:

CREATE TABLE persons (
    id int,
    first_name varchar(255),
    last_name varchar(255)
);

CREATE TABLE reservations (
    id int,
    person_id int,
    room_number int,
    check_in_date date,
    reserved_at datetime
);

INSERT INTO persons 
	(id, first_name, last_name) 
VALUES 
	(1, "Ivan", "Ivanov"), 
	(2, "Petr", "Petrov");
	
INSERT INTO reservations 
	(id, person_id, room_number, check_in_date, reserved_at) 
VALUES 
	(1, 1, 666, "2020-01-01", "2020-01-01 00:00:00"), 
	(2, 2, 666, "2023-01-01", "2023-01-01 00:00:00"), 
	(3, 1, 777, "2023-01-01", "2023-01-01 00:00:00"), 
	(4, 2, 777, "2023-01-01", "2023-01-01 00:00:00"); 

Результат запроса на тестовых данных:
+-------------+------------+-----------+
| room_number | first_name | last_name |
+-------------+------------+-----------+
|         666 | Petr       | Petrov    |
+-------------+------------+-----------+


INSERT INTO reservations 
	(id, person_id, room_number, check_in_date, reserved_at) 
VALUES 
	(5, 1, 999, "2020-01-01", "2020-01-01 00:00:00"), 
	(6, 2, 999, "2020-01-01", "2020-01-01 23:59:59");

Результат запроса на обновленных данных:
+-------------+------------+-----------+
| room_number | first_name | last_name |
+-------------+------------+-----------+
|         666 | Petr       | Petrov    |
|         999 | Ivan       | Ivanov    |
|         999 | Petr       | Petrov    |
+-------------+------------+-----------+

INSERT INTO reservations 
	(id, person_id, room_number, check_in_date, reserved_at) 
VALUES 
	(7, 1, 333, "2025-01-01", "2023-01-01 00:00:00");

Результат запроса на обновленных данных:
+-------------+------------+-----------+
| room_number | first_name | last_name |
+-------------+------------+-----------+
|         666 | Petr       | Petrov    |
|         999 | Ivan       | Ivanov    |
|         999 | Petr       | Petrov    |
+-------------+------------+-----------+
(всё верно - последняя запись не должна попасть в выборку, т.к. бронь на 2025 год, а нам нужны те кто "проживал")
</pre>