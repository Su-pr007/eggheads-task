# Тестовые задачи для компании EGGHEADS

Решение задач от компании [EGGHEADS](https://eggheads.solutions/)

## Задача
Задача предоставлена в виде [docx документа](./EGGHEADS_задачки%20backend(тестовое).docx). Сохранена рядом с этим файлом

## Решение тестовых заданий


### Задание 1
Опишите, какие проблемы могут возникнуть при использовании данного кода
```php
// ...
$mysqli = new mysqli("localhost", "my_user", "my_password", "world");
$id = $_GET['id'];
$res = $mysqli->query('SELECT * FROM users WHERE u_id='. $id);
$user = $res->fetch_assoc();
// ...
```

#### Решение
При использовании подобного кода есть вероятность SQL-инъекций. В строку запроса без каких-либо проверок подставляется значение из GET параметра "id".
Как решение, можно было бы использовать метод `prepare` у использованной библиотеки mysqli.

Также есть другие замечания:
1. Статические данные для соединения с базой в идеале нужно вынести в константы, чтобы управлять ими из одного места, и не искать в коде;
2. Ограничить запрашиваемые из таблицы поля, чтобы снизить передаваемые данные из mysql в php, и облегчить скрипт;
3. Добавить завершение соединения с базой. Хоть представленный код и просто пример, не стоит забывать закрывать подключение, чтобы уменьшить нагрузку на бд.


### Задание 2
Рефакторинг участка кода

```php
// ...
$questionsQ = $mysqli->query('SELECT * FROM questions WHERE catalog_id='. $catId);
$result = array();
while ($question = $questionsQ->fetch_assoc()) {
	$userQ = $mysqli->query('SELECT name, gender FROM users WHERE id='. (int)$question[‘user_id’]);
	$user = $userQ->fetch_assoc();
	$result[] = array('question'=>$question, 'user'=>$user);
	$userQ->free();
}
$questionsQ->free();
// ...
```

#### Решение
1. Убрать венгерскую нотацию из названий переменных;
2. Использовать метод prepare, вместо query, чтобы обезопасить скрипт от незапланированного поведения. Также добавить освобождение результата запроса и закрытие запросов;
3. Добавить приведение типов данных, где они могут отличаться
4. Добавить ограничение по запрашиваемым полям таблицы;
5. Заменить `array()` на `[]` (по PSR)


#### Итоговый вариант:

```php
// ...
$questionsQuery = $mysqli->prepare('SELECT `id`, `user_id` FROM questions WHERE `catalog_id` = ?');
$questionsQuery->bind_param('i', (int)$catId);
$questionsQuery->execute();
$questionsResult = $questionsQuery->get_result();

$result = [];
while ($question = $questionsResult->fetch_assoc()) {
	$userQuery = $mysqli->prepare('SELECT name, gender FROM users WHERE `id` = ?');
	$userQuery->bind_param('i', (int)$question[‘user_id’]);
	$userQuery->execute();
	$userResult = $userQuery->get_result();

	$user = $userResult->fetch_assoc();
	$result[] = [
		'question' => $question,
		'user' => $user
	];

	$userQ->free();
	$userQuery->close();
}
$questionsQuery->free();
$questionsQuery->close();
// ...
```

Замечания:
- Переменная `$question` добавляется в итоговый массив, поэтому не факт, что в запросе перечислены все нужные поля. В другом месте могут использоваться другие поля;
- Для генерации запросов лучше использовать ORM, или хотя бы собственные методы, чтобы избежать бессмысленного раздувания кода сырыми SQL запросами и методами их обработки.


### Задание 3

#### Задание:

Необходимо выбрать одним запросом список контрагентов в следующем формате (следует учесть, что будет включена опция only_full_group_by в MySql):
    1. Имя контрагента
    2. Его телефон
    3. Сумма всех его заказов
    4. Его средний чек
    5. Дата последнего заказа


#### Решение

```mysql
SELECT
    users.name `user_name`,
    users.phone `user_phone`,
    SUM(orders.subtotal) `total_sum`,
    AVG(orders.subtotal) `average_sum`,
    MAX(orders.created) `last_order_date`
FROM users
    JOIN orders
        ON users.id = orders.user_id
GROUP BY users.name, users.phone;
```


На всякий случай, сохранил код для генерации таблиц:
```mysql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    phone VARCHAR(20),
    email VARCHAR(255),
    created DATETIME
);

CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    subtotal INT,
    created DATETIME,
    city_id INT,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

INSERT INTO users (name, phone, email) VALUES
('Иванов Иван', '123-45-67', 'ivanov@test.com'),
('Петров Петр', '987-65-43', 'petrov@test.com');

INSERT INTO orders (subtotal, created, city_id, user_id) VALUES
(100, '2022-01-10 08:00:00', 1, 1),
(150, '2022-01-11 12:30:00', 2, 1),
(120, '2022-01-12 15:45:00', 1, 2);
```


### Задание 4

1. Написать запрос для вывода самой большой зарплаты в каждом департаменте
2. Написать запрос для вывода списка сотрудников из 3-го департамента у которых ЗП больше 90000
3. Написать запрос по созданию индексов для ускорения

#### Решение
1. Запрос:
```mysql
SELECT DepartmentId, MAX(Salary) FROM employees GROUP BY DepartmentId;
```
2. Запрос:
```mysql
SELECT Name, LastName FROM employees WHERE DepartmentId = 3 AND Salary > 90000;
```
3. Допустим, нам понадобился индекс для полей `Name` и `LastName`, чтобы ускорить по ним поиск.
Запрос:
```mysql
CREATE INDEX employeesNames ON employees (`Name`, `LastName`);
```

На всякий случай, сохранил код для генерации таблицы:
```mysql
CREATE TABLE employees (
    Id INT PRIMARY KEY AUTO_INCREMENT,
    Name VARCHAR(50),
    LastName VARCHAR(50),
    DepartmentId INT,
    Salary INT
);

INSERT INTO employees (Name, LastName, DepartmentId, Salary) VALUES
('Иван', 'Смирнов', 2, 100000),
('Максим', 'Петров', 2, 90000),
('Роман', 'Иванов', 3, 95000);
```


### Задание 5

Рефакторинг кода
```javascript
// ...
function printOrderTotal(responseString) {
   var responseJSON = JSON.parse(responseString);
   responseJSON.forEach(function(item, index){
      if (item.price = undefined) {
         item.price = 0;
      }
      orderSubtotal += item.price;
   });
   console.log( 'Стоимость заказа: ' + total > 0? 'Бесплатно': total + ' руб.');
}
// ...
```

#### Решение

1. Заменить `var` на `const`. `var` - устаревшее ключевое слово, которое может вызвать непредусмотренный функцией эффект, т.к. создаёт переменную выше области видимости функции. Мы не планируем в дальнейшем изменять значение переменной, поэтому можно создать её как константу;
2. Сравнение написано с одним знаком равенства. Если мы хотим положиться на нечёткую типизацию js, можно оставить в условии `if (!item.price)`, но если хотим придерживаться строгой типизации, то лучше проверять переменную с учётом типов (`if (item.price === undefined)`);
3. Переменная `orderSubtotal` нигде не создаётся. Следует добавить её инициализацию;
4. Можно заменить цикл forEach на блок `for(... of ...)`. Так будет быстрее, + в теории так будет затрачено меньше памяти, ведь не придётся создавать функции;
5. Переменная `total` не существует, но используется в выводе в `console.log`. Скорее всего, подразумевался вывод `orderSubtotal`

#### Результат

```javascript
// ...
function printOrderTotal (responseString) {
	const responseJSON = JSON.parse(responseString);
	let orderSubtotal = 0;

	for (let item of responseJSON) {
		if (item.price === undefined) {
			item.price = 0;
		}

		orderSubtotal += item.price;
	}

	console.log('Стоимость заказа: ' + orderSubtotal > 0 ? 'Бесплатно' : orderSubtotal + ' руб.');
}
// ...
```


#### Примечания
Метод `JSON.parse` может выбросить исключение, если переданную строку не удастся распарсить. Возможно, следует добавить в блок `try-catch`
