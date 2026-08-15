# 🐾 Go Pet Shop

**Go Pet Shop** — учебный проект интернет-магазина товаров для питомцев на Go с PostgreSQL.
Практика посвящена написанию unit-тестов для HTTP-хендлеров с использованием ручного мока (v1) и автогенерируемого мока через mockery (v2).

---

## ⚙️ Технологии

- **Go 1.24+** — основной язык разработки (нужен для `tool`-зависимостей в `go.mod`, используемых веткой v2)
- **PostgreSQL** — база данных
- **chi** — HTTP-роутер
- **golang-migrate** — управление миграциями БД
- **testify** — assertions и моки
- **mockery v3** — генерация моков из интерфейсов
- **golangci-lint** — статический анализ кода
- **Taskfile** — автоматизация команд

---

## 🧩 Версии проекта

Проект разделён на две ветки, каждая демонстрирует свой подход к мокированию зависимостей в тестах.

### `v1` — ручной мок

Мок интерфейса `Products` (`internal/handlers/product/product_mock.go`) написан вручную.
Поведение методов задаётся через поля-функции:

```go
mock := &ProductsMock{
    GetAllProductsFunc: func(ctx context.Context) ([]models.Product, error) {
        return nil, errors.New("db error")
    },
}
```

**Плюс:** просто и наглядно для понимания, как работает подмена зависимостей через интерфейсы.
**Минус:** мок нужно поддерживать руками при любом изменении интерфейса `Products`.

### `v2` — mockery + testify

Мок генерируется автоматически через [mockery v3](https://github.com/vektra/mockery) на основе интерфейса `Products`, конфиг — `.mockery.yml`.

Над интерфейсом добавлена директива генерации (`internal/handlers/product/products.go`):

```go
//go:generate go run github.com/vektra/mockery/v3

type Products interface {
    GetAllProducts(ctx context.Context) ([]models.Product, error)
    CreateProduct(ctx context.Context, product models.Product) (int, error)
    DeleteProduct(ctx context.Context, id int) error
    UpdateProduct(ctx context.Context, product models.Product) error
}
```

Сгенерировать (или перегенерировать после изменения интерфейса) мок:

```bash
go generate ./...
```

Мок основан на `testify/mock` и настраивается через типобезопасный `.EXPECT()`:

```go
mock := NewProductsMock(t)
mock.EXPECT().GetAllProducts(mock2.Anything).Return(nil, errors.New("db error"))
```

Также `NewProductsMock(t)` регистрирует `t.Cleanup`, который в конце теста вызывает `AssertExpectations` — если хендлер не сделал ожидаемый вызов к storage, тест упадёт сам, без ручных проверок.

**Плюс:** мок регенерируется автоматически при изменении интерфейса одной командой, без ручной установки бинарника — `mockery/v3` подключён как [tool-зависимость](https://go.dev/doc/modules/managing-dependencies#tools) в `go.mod` (`go get -tool github.com/vektra/mockery/v3`), поэтому при клонировании репозитория `go generate ./...` скачает и запустит нужную версию mockery самостоятельно.

**Минус:** требует понимания API testify/mock (`.EXPECT()`, `.On()`, `.Return()`), и первый запуск `go generate` требует интернет-соединения для скачивания инструмента.

### Ключевое отличие v1 vs v2

| | v1 (ручной мок) | v2 (mockery) |
|---|---|---|
| Кто пишет мок | вы вручную | генерируется командой `go generate ./...` |
| Настройка поведения | через поля-функции (`XxxFunc`) | через `.EXPECT()` / `.On()` |
| Проверка «метод точно вызвался» | нет автоматической | есть (`AssertExpectations`) |
| Поддержка при росте интерфейса | руками | перегенерировать и всё |

---

## 🧪 Тестирование

Тесты покрывают HTTP-хендлеры `product` для сценариев:

| Хендлер | Success (200) | BadRequest (400) | Fail (500) |
|---|---|---|---|
| GetAllProducts | ✅ | — | ✅ |
| CreateProduct | ✅ | ✅ | ✅ |
| UpdateProduct | ✅ | ✅ | ✅ |
| DeleteProduct | ✅ | ✅ | ✅ |

Запуск всех тестов:

```bash
go test ./...
```

С подробным выводом:

```bash
go test -v ./...
```

---

## 🛠 Как запустить проект

1. Клонировать репозиторий:

   ```bash
   git clone https://github.com/Che-Misty/go-pet-shop.git
   cd go-pet-shop
   ```

2. Создать `.env` по примеру `.env.example`, указать строку подключения к БД:

   ```
   DATABASE_URL=postgres://user:password@localhost:5432/petshop?sslmode=disable
   ```

3. Запуск в Docker (рекомендуется):

   ```bash
   docker-compose up -d
   ```

4. Запуск локально (нужен установленный PostgreSQL):

   ```bash
   task migrate
   go run cmd/app/main.go
   ```

5. API будет доступно по адресу `http://localhost:8080`

---

## 🔍 Линтер

```bash
task linter
```

Проверки: `unused`, `bodyclose`, `govet`, `staticcheck`, `errcheck`, `ineffassign`, `gocyclo`.

---

## 📂 Структура проекта

```
go-pet-shop/
├── cmd/                        # Точки входа (app — сервер, migrator — миграции)
├── config/                     # Конфигурация (yaml/json)
├── internal/
│   ├── config/                 # Загрузка и валидация конфигурации
│   ├── handlers/
│   │   └── product/            # HTTP-хендлеры, мок и тесты для товаров
│   └── storage/                # Работа с БД
├── migrations/                 # SQL-файлы миграций
├── models/                     # Доменные модели и DTO
├── .env                        # Переменные окружения (не хранится в git)
├── .mockery.yml                # Конфигурация mockery (только в v2)
├── .golangci.yaml               # Настройки линтера
├── Taskfile.yaml                # Скрипты для работы с проектом
├── go.mod                       # Модуль Go
└── README.md                    # Документация
```