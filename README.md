# ЭФМО-01-25 Буров М.А. ПР14

# Описание проекта
Оптимизация запросов к БД. Использование connection pool

# Требования к проекту
* Go 1.25+
* PostgreSQL
* Git

# Версия Go
<img width="317" height="55" alt="image" src="https://github.com/user-attachments/assets/43f9087b-95b9-4c7d-86e9-746258c45c63" />

# Цели:
1.	Научиться находить «узкие места» в SQL-запросах и устранять их (индексы, переписывание запросов, пагинация, батчинг).
2.	Освоить настройку пула подключений (connection pool) в Go и параметры его тюнинга.
3.	Научиться использовать EXPLAIN/ANALYZE, базовые метрики (pg_stat_statements), подготовленные запросы и транзакции.
4.	Применить техники уменьшения N+1 запросов и сокращения аллокаций на горячем пути.

# Структура проекта
Дерево структуры проекта: 
```
pz14-notes-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── core/
│   │   ├── note.go
│   │   └── types.go
│   ├── http/
│   │   ├── router.go
│   │   └── handlers/
│   │       └── notes.go
│   ├── db/
│   │   └── connection.go
│   └── repo/
│       ├── interface.go
│       ├── note_postgres.go
│       └── note_mem.go
├── docker-compose.yml
├── go.mod
└── go.sum
```

# Скриншоты

Индексы

<img width="579" height="390" alt="image" src="https://github.com/user-attachments/assets/325fff6d-b452-4c9b-be27-3ae9e91e8caf" />

Неоптимизированная пагинация с OFFSET

<img width="1227" height="245" alt="image" src="https://github.com/user-attachments/assets/a008d212-abcf-447b-b6e5-18e8370263fb" />

Множественные одиночные запросы (N+1)

<img width="514" height="429" alt="image" src="https://github.com/user-attachments/assets/3a56d9dd-983a-45ed-a528-f0ed803b833c" />

Keyset-пагинация (оптимизированная)

<img width="1225" height="264" alt="image" src="https://github.com/user-attachments/assets/06c20ffe-3a71-46f7-ab87-8863f88460fc" />

Батчинг вместо N+1

<img width="1110" height="263" alt="image" src="https://github.com/user-attachments/assets/38314ec7-4af1-4f41-bf86-12129a06e50b" />

## Нагрузочное тестирование

Keyset-пагинация

```
hey -n 1000 -c 100 http://localhost:8080/api/v1/notes\?page_size=20
```

<img width="909" height="838" alt="image" src="https://github.com/user-attachments/assets/9f94503e-235f-42a2-a31c-5f71d5c17173" />

Батчинг

```
hey -n 1000 -c 100 http://localhost:8080/api/v1/notes/batch\?ids=1,2,3,4
```

<img width="962" height="821" alt="image" src="https://github.com/user-attachments/assets/6152bee6-125b-4301-90a5-1cb06a537e63" />

# Краткие выводы

В ходе работы были применены три ключевые техники оптимизации: создание индексов на часто используемых столбцах (B-Tree, GIN, покрывающие), переход с OFFSET/LIMIT на keyset-пагинацию и замена N+1 запросов на батчинг. Настройка connection pool (MaxOpenConns=25, MaxIdleConns=12) обеспечила баланс между пропускной способностью и нагрузкой на БД.

# Ответы на контрольные вопросы

1.	Чем keyset-пагинация лучше OFFSET/LIMIT на больших объёмах?

OFFSET требует сканирования всех N смещаемых строк (O(n)), тогда как keyset использует условие WHERE (created_at, id) < (...) и индекс для прямого доступа к нужным строкам (O(limit)). На больших смещениях (страница 1000+) keyset работает в 100-1000 раз быстрее, так как OFFSET 1000000 сканирует 1000020 строк, а keyset — ровно 20. Это константная сложность, не зависящая от позиции в наборе данных.

2.	Когда нужен покрывающий индекс и чем он отличается от обычного?

Покрывающий индекс (INCLUDE) пакует все нужные для SELECT поля прямо в индекс, избегая обращения к основной таблице (Index-Only Scan). Обычный индекс требует двух операций: поиск по индексу, затем загрузка строк из таблицы (Heap Fetch). Покрывающий нужен для часто используемых SELECT-ов на подмножество столбцов в большие таблицы (>1M записей), уменьшая I/O и ускоряя поиск на 2-5 раз.

3.	Какие параметры пула подключений в Go вы настраиваете и почему?

- MaxOpenConns (обычно 20-30) ограничивает одновременные соединения, предотвращая перегрузку БД;
- MaxIdleConns (50% от MaxOpenConns) держит готовые соединения, экономя на переподключениях;
- ConnMaxLifetime (30 минут) ротирует соединения для очистки памяти на сервере;
- ConnMaxIdleTime (5-10 минут) закрывает неиспользуемые соединения. 

4.	Что показывает EXPLAIN (ANALYZE, BUFFERS) и как отличить Seq Scan от Index Scan?

EXPLAIN показывает план выполнения (cost, rows); ANALYZE добавляет фактические значения (actual rows, execution time); BUFFERS показывает I/O (shared hit/read). Seq Scan сканирует все строки таблицы (медленно на больших таблицах, используется когда нет индекса или WHERE неселективен), тогда как Index Scan находит строки через индекс (быстро, используется for WHERE с индексом). В выводе: Seq Scan имеет высокие Buffers и execution time, Index Scan — низкие. Index-Only Scan (самый быстрый) вообще не обращается к таблице.

5.	Как устранить N+1 запросов? Приведите 2 способа.

Способ 1 (батчинг): вместо цикла с запросами SELECT * FROM notes WHERE id = $1 (повторяется N раз), выполнить один запрос SELECT * FROM notes WHERE id = ANY($1) с массивом всех ID, потом связать результаты в приложении. 

Способ 2 (JOIN): заменить цикл на один SQL-запрос с LEFT JOIN, чтобы получить всю информацию сразу: SELECT n.*, a.name FROM notes n LEFT JOIN authors a ON n.author_id = a.id. Батчинг гибче, JOIN проще для стандартных связей.

6.	Когда уместны prepared statements и какие плюсы они дают?

Prepared statements уместны для запросов, выполняемых много раз (циклы, батчинг), и для критичных по производительности эндпоинтов. Плюсы: парсинг и планирование выполняются один раз, параметры повторно используют кэшированный план (-20-30% latency); защита от SQL-injection (параметры автоматически экранируются); в pgx подготовка часто неявная (кэш после 5-10 вызовов). Не используйте для одноразовых запросов, так как overhead на подготовку > выгода.

7.	Как выбрать «правильный» размер пула для сервиса и БД? Какие метрики смотреть?

Базовая формула: MaxOpenConns = CPU_cores × 3-4 (обычно 20-30), MaxIdleConns = 50% MaxOpenConns. 

Ключевые метрики: pool utilization 50-80% (goal), queue wait latency < 10ms (warning > 100ms), connection errors (warning > 0.1%), p95/p99 latency (цель < 50/200ms). 

Слишком маленький пул = очереди и таймауты, слишком большой = context switching на БД. На практике ищите точку, где p99 latency стабилизируется, а ошибок нет.
