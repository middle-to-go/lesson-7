---
customTheme : "presentation"

---

## Лекция 7. Работа с БД в Go
### OZON

### Москва, 2021


---

### Лекции

1. <span style="color:gray">Введение. Рабочее окружение. Структура программы. Инструментарий.</span>
2. <span style="color:gray">Базовые конструкции и операторы. Встроенные типы и структуры данных.</span>
3. <span style="color:gray">Структуры данных, отложенные вызовы, обработка ошибок и основы тестирования</span>
4. <span style="color:gray">Интерфейсы, моки и тестирование с ними</span>
5. <span style="color:gray">Асинхронные сущности и паттерны в Go</span>
6. <span style="color:gray">Protobuf и gRPC</span>
7. Работа с БД в Go
8. <span style="color:gray">Брокеры сообщений. Трассировка. Метрики. </span>

---

### Темы

Сегодня мы поговорим про:

1. Установка и работа с PostgreSQL
1. Подключение к СУБД и настройка пула подключений <!-- .element: class="fragment" -->
1. Выполнение запросов и получение результатов <!-- .element: class="fragment" -->
1. Стандартные интерфейсы sql.DB, sql.Rows, sql.Tx <!-- .element: class="fragment" -->
1. Использование транзакций <!-- .element: class="fragment" -->
1. SQL инъекции и борьба с ними <!-- .element: class="fragment" -->
1. Подводные камни при работе с БД <!-- .element: class="fragment" -->
1. Интеграция БД в микросервис <!-- .element: class="fragment" -->
1. Тестирование <!-- .element: class="fragment" -->

---

### Обозначения

* 📽️ - посмотри воркшоу
* ⚗️ - проведи эксперимент
* 🔬 - изучи внимательно
* 📖 - прочитай документация
* 🪙 - подумай о сложности
* 🐞 - запомни ошибку
* 🔨 - запомни решение
* 🏔️ - обойди камень предкновенья
* ⏰ - сделай перерыв
* 🏡 - попробуй дома
* 💡 - обсуди светлые идеи
* 🙋 - задай вопрос
* ⚡ - запомни панику

---

### Установка PostgreSQL локально

Устанавливаем сервер из консоли (пример для Ubuntu):

```bash
# обновить пакеты
$ sudo apt-get update
# установить PostgreSQL сервер и клиент
$ sudo apt-get -y install postgresql
# запустить PostgreSQL
$ sudo systemctl start postgresql
# подключиться под пользователем, созданным по умолчанию
$ sudo -u postgres psql
```

https://www.postgresql.org/download/linux/ubuntu/

---

### Работаем с PostgreSQL локально через Docker

Запускаем контейнер с сервером PostgreSQL:

```bash
  docker run -d \
  --name pg \
  -e POSTGRES_PASSWORD=postgres \
  -e PGDATA=/var/lib/postgresql/data/pgdata \
  -v /Users/sivliev/psqldata:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres
 ```

Ждём немного, пока СУБД поднимется
Подключаемся к серверу:
```bash
docker exec -it pg psql -Upostgres -dpostgres
```

https://hub.docker.com/_/postgres


---

### Работаем с PostgreSQL локально через Docker

Можно описать в docker-compose:

```yaml
version: '3.1'

services:

  db:
    image: postgres
    restart: always
    volumes:
      - /Users/sivliev/psqldata:/var/lib/postgresql/data 
    environment:
      POSTGRES_PASSWORD: postgres
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - 5432:5432
  adminer:
    image: adminer
    restart: always
    ports:
      - 8080:8080
```

```bash
docker-compose up
```
 
N.b.: в последних версиях Docker `docker-compose` стал частью команды `docker` и вызывается как `docker compose`.


---

### Создание БД и пользователя

```sql
postgres=# create database lecture_7_demo;
postgres=# create user sirius with encrypted password 'lecture_7_demo';
postgres=# grant all privileges on database lecture_7_demo to sirius;
```

---

### Миграции через Goose

https://github.com/pressly/goose

Goose — менеджер миграции базы данных общего назначения. 

Идея проста:
* Вы предоставляете файлы схемы SQL, которые следуют определенному соглашению об именах
* Вы предоставляете простой `dbconf.yml` файл, который говорит Goose, как подключиться к вашим различным базам данных
* Goose предоставляет вам простые инструменты для обновления схемы (`goose up`), проверки (`goose status`) и даже возврата (`goose down`).


---

### Пример миграции

```sql

-- +goose Up
CREATE table books (
  id serial primary key,
  title text,
  description text,
  meta jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz
);

INSERT INTO books (title, description, meta, updated_at)
VALUES
  ('Мастер и Маргарита', 'test description 1', '{}', now()),
  ('Граф Монте-Кристо', 'test description 2', null, null),
  ('Марсианин', 'test description 3', '{"author": "Энди Вейер"}', now());

-- +goose Down
drop table books;
```

---

### Подключение к PostgreSQL из Go, простейший вариант


Создание подключения:

```go
import "database/sql"
import _ "github.com/jackc/pgx/stdlib"

dsn := "..."
db, err := sql.Open("pgx", dsn) // *sql.DB
if err != nil {
 log.Fatalf("failed to load driver: %v", err)
}
// создан пул соединений
```

Использование подключения:

```go
err := db.PingContext(ctx)
if err != nil {
 return xerrors.Errorf("failed to connect to db: %v", err)
}
// работаем с db
```

http://go-database-sql.org/importing.html

http://go-database-sql.org/accessing.html


---

### DataSourceName

DSN - строка подключения к базе, содержит все необходимые опции.
Синтаксис DSN зависит от используемой базы данных и драйвера.

Например для PostgreSQL:

```
"postgres://myuser:mypass@localhost:5432/mydb?sslmode=verify-full"
```

или

```
"user=myuser dbname=mydb sslmode=verify-full password=mypass"
```

---

### DataSourceName

Параметры: 
* `host` - Сервер базы данных или путь к UNIX-сокету (по-умолчанию `localhost`)
* `port` - Порт базы данных (по-умолчанию `5432`)
* `dbname` - Имя базы данных
* `user` - Пользователь в СУБД (по умолчанию - пользователь OS)
* `password` - Пароль пользователя

Подробнее: https://godoc.org/github.com/lib/pq

---

### Пул соединений

🙋 А зачем нам вообще пул соединений?

`sql.DB` - это пул соединений с базой данных. Соединения будут открываться по мере необходимости.

`sql.DB` - безопасен для конкурентного использования (так же как `http.Client`)

Настройки пула:

```go
// Макс. число открытых соединений от этого процесса
db.SetMaxOpenConns(n int)
// Макс. число открытых неиспользуемых соединений
db.SetMaxIdleConns(n int)
// Макс. время жизни одного подключения
db.SetConnMaxLifetime(d time.Duration)
```

http://go-database-sql.org/connection-pool.html

---

### Пул соединений

<img src="images/connection_pool.png"></img>

---

### Выполнение запросов

```go
query := `insert into events(owner, title, descr, start_date, end_date)
 values($1, $2, $3, $4, $5)`
result, err := db.ExecContext(ctx, query,
 42, "new year", "watch the irony of fate", "2019-12-31", "2019-12-31"
) // sql.Result
if err != nil {
 // обработать ошибку
}
// Авто-генерируемый ID (SERIAL)
eventId, err := result.LastInsertId() // int64
// Количество измененных строк
rowsAffected, err := result.RowsAffected() // int64
```
http://go-database-sql.org/retrieving.html

---

### Получение результатов

```go
query := `
 select id, title, descr
 from events
 where owner = $1 and start_date = $2
`
rows, err := db.QueryContext(ctx, query, owner, date)
if err != nil {
  // ошибка при выполнении запроса
}
defer rows.Close()
for rows.Next() {
  var id int64
  var title, descr string
  if err := rows.Scan(&id, &title, &descr); err != nil {
  // ошибка сканирования
  }
  // обрабатываем строку
  fmt.Printf("%d %s %s\n", id, title, descr)
}
if err := rows.Err(); err != nil {
  // ошибка при получении результатов
}
```

---

### Объект sql.Rows

```go
// возвращает имена колонок в выборке
rows.Columns() ([]string, error)
// возвращает типы колонок в выборке
rows.ColumnTypes() ([]*ColumnType, error)
// переходит к следующей строке или возвращает false
rows.Next() bool
// заполняет переменные из текущей строки
rows.Scan(dest ...interface{}) error
// закрывает объект Rows
rows.Close()
// возвращает ошибку, встреченную при итерации
rows.Err() error
```

---

### Получение одной строки

```go
query := "select * from events where id = $1"
row := db.QueryRowContext(ctx, query, id)
var id int64
var title, descr string
err := row.Scan(&id, &title, &descr)
if err == sql.ErrNoRows {
  // строки не найдено
} else if err != nil {
  // "настоящая" ошибка
}
```

---

### PreparedStatements

PreparedStatement - это заранее разобранный запрос, который можно выполнять повторно.

PreparedStatement - временный объект, который создается в СУБД и живет в рамках сессии, или пока не будет закрыт.

```go

// создаем подготовленный запрос
stmt, err := db.Prepare("delete from events where id = $1") // *sql.Stmt
if err != nil {
 log.Fatal(err)
}
// освобождаем ресурсы в СУБД
defer stmt.Close()
// многократно выполняем запрос
  for _, id := range ids {
  _, err := stmt.Exec(id)
  if err != nil {
    log.Fatal(err)
  }
}

```

http://go-database-sql.org/prepared.html

---

### Работа с соединением

`*sql.DB` - это пул соединений. Даже последовательные запросы могут использовать разные соединения с базой.

Если нужно получить одно конкретное соединение, то

```go
conn, err := db.Conn(ctx) // *sql.Conn
// вернуть соединение в pool
defer conn.Close()
// далее - обычная работа как с *sql.DB
err := conn.ExecContext(ctx, query1, arg1, arg2)
rows, err := conn.QueryContext(ctx, query2, arg1, arg2)
```

---

### Транзакции

Транзакция - группа запросов, которые либо выполняются, либо не выполняются вместе. Внутри транзакции все запросы видят "согласованное" состояние данных.

На уровне SQL для транзакций используются отдельные запросы: `BEGIN`, `COMMIT`,`ROLLBACK`.

```go
tx, err := db.BeginTx(ctx, nil) // *sql.Tx
if err != nil {
  log.Fatal(err)
}
// далее - обычная работа как с *sql.DB
err := tx.ExecContext(ctx, query1, arg1, arg2)
rows, err := tx.QueryContext(ctx, query2, arg1, arg2)
err := tx.Commit() // или tx.Rollback()
if err != nil {
  // commit не прошел, данные не изменились
}
  // далее объект tx не пригоден для использования
```

http://go-database-sql.org/modifying.html

---

### Обертка над транзакциями

```go

// WithTxFunc custom type of func to helper
type WithTxFunc func(ctx context.Context, tx sqlutil.Tx) error

// WithTx runs code with transaction
func (d *tx) WithTx(ctx context.Context, fn WithTxFunc) error {
  t, err := d.db.Begin(ctx, nil)
  if err != nil {
    return errors.Wrap(err, "Tx.Begin")
  }
  if err = fn(ctx, t); err != nil {
    if errRollback := t.Rollback(); errRollback != nil {
      return errors.Wrap(err, "Tx.Rollback")
    }
    return errors.Wrap(err, "Tx.WithTxFunc")
  }
  if err = t.Commit(); err != nil {
    return errors.Wrap(err, "Tx.Commit")
  }
  return nil
}
```


---

### Обертка над транзакциями

Определены у `*sql.DB`, `*sql.Conn`, `*sql.Tx`, `*sql.Stmt`:

```go
// изменение данных
ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error)
// получение данных (select)
QueryContext(ctx context.Context, query string, args ...interface{}) (*Rows, error)
// получение одной строки
QueryRowContext(ctx context.Context, query string, args ...interface{}) *Row
```

🐞 Внимание, ошибка:
```go
_, err := db.QueryContext(ctx, "delete from events where id = $1", 42)
```

---

### NULL


В SQL базах любая колонка может быть объявлена к NULL / NOT NULL. NULL - это не 0 и не пустая строка, это отсутствие значения.

```sql
create table users (
  id serial primary key,
  name text not null,
  age int null
);
```

Для обработки `NULL` в Go предлагается использовать специальные типы:

```go
var id, realAge int64
var name string
var age sql.NullInt64
err := db.QueryRowContext(ctx, "select * from users where id = 1").Scan(&id, &name, &age)

if age.Valid {
  realAge = age.Int64
} else {
  // обработка на ваше усмотрение
}
```

---

### SQL Injection

<img src="images/exploits_of_a_mom.png">

---

### SQL Injection

Опасно:
```go
query := "select * from users where name = '" + name + "'"
query := fmt.Sprintf("select * from users where name = '%s'", name)
```

Потому что в name может оказаться что-то типа:
```
"jack'; truncate users; select 'pawned"
```


---

### SQL Injection

Правильный подход - использовать placeholders для подстановки значений в SQL:
```go
row := db.QueryRowContext(ctx, "select * from users where name = $1", name)
```

Однако это не всегда возможно. Так работать не будет:
```go
db.QueryRowContext(ctx, "select * from $1 where name = $2", table, name)
db.QueryRowContext(ctx, "select * from user order by $1 limit 3", order)
```

Проверить код на инъекции (и другие проблемы безопасности): https://github.com/securego/gosec

---

### Squirrel - fluent SQL generator for Go

```go
sql, args, err := sq.
 Insert("users").Columns("name", "age").
 Values("moe", 13).Values("larry", sq.Expr("? + 5", 12)).
 ToSql()
```

```
sql == "INSERT INTO users (name,age) VALUES (?,?),(?,? + 5)"
```

---

### Squirrel - fluent SQL generator for Go

```go

query := sq.Insert(tableName).
  Columns("description").
  Values(task.Description).
  Suffix("RETURNING \"id\"").
  RunWith(r.db).
  PlaceholderFormat(sq.Dollar)
​
query.QueryRowContext(ctx).Scan(&task.Id)
```

```go
  query := sq.Update(tableName).
    SetMap(map[string]interface{}{
      "id": task.Description,
    }).
    Where(sq.Eq{"id": task.Id}).
    RunWith(r.db).
    PlaceholderFormat(sq.Dollar)
​
  _, err := query.ExecContext(ctx)
  return err
```


---

### Проблемы database/sql

* placeholder зависят от базы: ( `$1` в Postgres, `?` в MySQL, `:name` в Oracle)
* Есть только базовые типы, но нет, например `sql.NullDate`
* `rows.Scan(arg1, arg2, arg3)` - неудобен, нужно помнить порядок и типы колонок.
* Нет возможности `rows.StructScan(&event)`

---

### Расширение sqlx

`jmoiron/sqlx` - обертка, прозрачно расширяющая стандартную библиотеку `database/sql`:
* `sqlx.DB` - обертка над `*sql.DB`
* `sqlx.Tx` - обертка над `*sql.Tx`
* `sqlx.Stmt` - обертка над `*sql.Stmt`
* `sqlx.NamedStmt` - PreparedStatement с поддержкой именованых параметров

---

### Расширение sqlx

Подключение `jmoiron/sqlx`:

```go
import "github.com/jmoiron/sqlx"


db, err := sqlx.Open("pgx", dsn) // *sqlx.DB
rows, err := db.QueryContext("select * from events") // *sqlx.Rows
...
```

---

### sqlx: именованные placeholder'ы

Можно передавать параметры запроса в виде словаря:

```go
sql := "select * from events where owner = :owner and start_date = :start"
rows, err := db.NamedQueryContext(ctx, sql, map[string]interface{}{
  "owner": 42,
  "start": "2019-12-31",
})
```
Или структуры:

```go
type QueryArgs{
 Owner int64
 Start string
}

sql := "select * from events where owner = :owner and start_date = :start"

rows, err := db.NamedQueryContext(ctx, sql, QueryArgs{
 Owner: 42,
 Start: "2019-12-31",
})
```

---

### sqlx: сканирование

Можно сканировать результаты в словарь:


```go

sql := "select * from events where start_date > $1"
rows, err := db.QueryContext(ctx, sql, "2020-01-01") // *sqlx.Rows
for rows.Next() {
 results := make(map[string]interface{})
 err := rows.MapScan(results)
 if err != nil {
  log.Fatal(err)
 }
 // обрабатываем result
}
```

---

### sqlx: сканирование

Можно сканировать результаты в структуру:

```go
type Event {
  Id int64
  Title string
  Description string `db:"descr"`
}
sql := "select * from events where start_date > $1"
rows, err := db.NamedQueryContext(ctx, sql, "2020-01-01") // *sqlx.Rows
events := make([]Event)
for rows.Next() {
    var event Event
    err := rows.StructScan(&event)
    if err != nil {
    log.Fatal(err)
  }
  events = append(events, event)
}
```

---

### Драйверы для Postgres

* Лучший драйвер на текущий момент: https://github.com/jackc/pgx
* Другой часто используемый драйвер (менее производительный): https://github.com/lib/pq

---

### ORM

* https://gorm.io/ - использует пустые интерфейсы :(
* https://github.com/go-reform/reform - использует кодогенерацию, но разработка немного заброшена

---

### Тестирование

https://github.com/DATA-DOG/go-txdb -- Single transaction based sql.Driver for GO


```go
func init() {
  txdb.Register("txdb", "mysql", "root@/txdb_test")
}

func main() {
  db, err := sql.Open("txdb", "identifier")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()
  if _, err := db.Exec(`INSERT INTO users(username) VALUES("gopher")`); err != nil {
    log.Fatal(err)
    }
 }
 ```

---

### Моки

https://github.com/DATA-DOG/go-sqlmock

```go

func TestShouldUpdateStats(t *testing.T) {
  db, mock, err := sqlmock.New()
  if err != nil {
    t.Fatalf("an error '%s' was not expected when opening a stub database connection", err)
  }
  defer db.Close()
  mock.ExpectBegin()
  mock.ExpectExec("INSERT INTO product_viewers").WithArgs(2, 3)).WillReturnResult(sqlmock.NewResult(1, 1))
  mock.ExpectCommit()
  if err = recordStats(db, 2, 3); err != nil {
    t.Errorf("error was not expected while updating stats: %s", err)
  }
  if err := mock.ExpectationsWereMet(); err != nil {
    t.Errorf("there were unfulfilled expectations: %s", err)
  }
}
```

---

### Фикстуры

https://github.com/go-testfixtures/testfixtures

```yaml
# comments.yml
- id: 1
 post_id: 1
 content: A comment...
 author_name: John Doe
 author_email: john@doe.com
 created_at: 2020-12-31 23:59:59
 updated_at: 2020-12-31 23:59:59
- id: 2
 post_id: 2
 content: Another comment...
 author_name: John Doe
 author_email: john@doe.com
 created_at: 2020-12-31 23:59:59
 updated_at: 2020-12-31 23:59:59
# ...
```

---

### Фикстуры

```go
fixtures, err := testfixtures.New(
  testfixtures.Database(db),
  testfixtures.Dialect("postgres"),
  testfixtures.Paths(
    "fixtures/orders.yml",
    "fixtures/customers.yml",
    "common_fixtures/users"
  ),
)
if err != nil {
 ...
}
```

---

### Подводные камни работы с БД

Хороший доклад, но больше про архитектурные вещи: https://www.youtube.com/watch?v=HjLnY0aPQZo

Большой список ошибок: https://stackoverflow.com/questions/621884/database-development-mistakes-made-by-application-developers

Понять, как работает ваш запрос поможет ```EXPLAIN ANALYZE <ваш запрос>```. 

Следите, что запрос не требует много времени, потребляет существующие индексы.

---

### Подводные камни работы с БД

От меня:

1. LIMIT и OFFSET 
2. IN по большому списку
3. Хранимые процедуры, создающие неочевидные связи в данных
4. GIN индекс над JSON полем
5. Драматическое разрастание таблиц за короткое время
6. Деление на горячие (UPDATE) и холодные данные (APPEND-ONLY)
7. SELECT FOR UPDATE

---

### Интеграция БД в микросервис

Нам требуется, чтобы у нас был один и тот же пул соединений в рамках сервиса.

Мы можем связать БД с контекстом.

```go

package db

import (
	"context"
	"log"

	_ "github.com/jackc/pgx/v4"
	_ "github.com/jackc/pgx/v4/stdlib"
	"google.golang.org/grpc"

	"database/sql"
)

var dbKey = "db"

// GetDBKey ...
func GetDBKey() string {
	return dbKey
}

// Connect ...
func Connect(DSN string) *sql.DB {
	db, err := sql.Open("pgx", DSN)
	if err != nil {
		log.Fatalf("connect do db error %v", err)
	}
	return db

}

```

---

### Интеграция БД в микросервис

```go
// NewContext ...
func NewContext(ctx context.Context, db *sql.DB) context.Context {
	ctxDB := context.WithValue(ctx, &dbKey, db)

	return ctxDB
}

// FromContext ...
func FromContext(ctx context.Context) *sql.DB {
	client, ok := ctx.Value(&dbKey).(*sql.DB)
	if !ok {
		panic("Error getting connection from context")
	}
	return client
}

// GetDB ...
func GetDB(ctx context.Context) *sql.DB {
	return FromContext(ctx)
}

// NewInterceptorWithDB ...
func NewInterceptorWithDB(db *sql.DB) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, 
  handler grpc.UnaryHandler) (resp interface{}, err error) {
		return handler(NewContext(ctx, db), req)
	}
}
```

---

### Интеграция БД в микросервис

Добавляем интерсептор.

```go
func run(dbConn *sql.DB) error {
	listen, err := net.Listen("tcp", grpcPort)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := grpc.NewServer(grpc.ChainUnaryInterceptor(db.NewInterceptorWithDB(dbConn)))
	desc.RegisterLecture6DemoServer(s, api.NewLecture6DemoAPI())



	if err := s.Serve(listen); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}

	return nil
}
```

---

### Интеграция БД в микросервис

```go
func (a *Lecture6DemoAPI)AddV1(ctx context.Context,
	req *desc.AddRequestV1) (*emptypb.Empty, error) {
	_, err := db.FromContext(ctx).Query("INSERT INTO tasks (task_id, text, callback_url) VALUES ($1, $2, $3)",
      req.ID, req.Text, req.CallbackURL)
	return &emptypb.Empty{}, err
}
```


---

### Другие ресурсы для изучения

* [ru] https://habr.com/ru/company/oleg-bunin/blog/461935/
* [en] http://go-database-sql.org/index.html
* [en] https://golang.org/pkg/database/sql
* [en] https://jmoiron.github.io/sqlx