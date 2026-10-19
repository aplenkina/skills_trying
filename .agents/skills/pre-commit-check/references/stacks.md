# Команды и ловушки по стекам

Это умолчания. Если проект задаёт свои команды (Makefile, scripts в `composer.json` или `package.json`, CI), бери их.

## Содержание
- Symfony / PHP в Docker
- Go
- Docker Compose
- Frontend / Python
- Секреты и мусор в диффе
- Smoke-проверка приложения
- Ловушки

## Symfony / PHP в Docker

Рабочий каталог сервиса обычно `/var/www/html/app`, пути к файлам внутри контейнера считай от него (у файла `app/src/Foo.php` в репозитории внутри контейнера путь `src/Foo.php`).

Форматтер, только изменённые файлы:
```bash
docker compose exec app vendor/bin/php-cs-fixer fix --config=.php-cs-fixer.dist.php --path-mode=intersection -- src/Foo.php tests/FooTest.php
docker compose exec app vendor/bin/php-cs-fixer fix --config=.php-cs-fixer.dist.php --path-mode=intersection --dry-run --diff -- src/Foo.php tests/FooTest.php
```
`--config` обязателен, когда путей несколько: без него fixer отвечает «For multiple paths config parameter is required». Предупреждение о версии PHP безвредно.

Линтеры (запускай те, что установлены):
```bash
docker compose exec -T app php bin/console lint:twig templates
docker compose exec -T app php bin/console lint:yaml config
docker compose exec -T app php bin/console lint:container
docker compose exec -T app php bin/console doctrine:schema:validate
docker compose exec -T app php bin/console doctrine:migrations:status
docker compose exec -T app composer validate --no-check-publish
```
Флаг `-T` нужен вне терминала. У консоли Symfony вывод цветной и многострочный: итог ищи так: `... | sed 's/\x1b\[[0-9;]*m//g' | grep -E "OK|ERROR"`, не по `tail -1` (последняя строка бывает пустой).
`composer validate` нужен, если менялся `composer.json`.

Тесты:
```bash
docker compose exec app php bin/phpunit
```
Если в `phpunit.dist.xml` есть `failOnNotice`/`failOnWarning`, то мок без ожиданий даёт notice и валит прогон: для заглушек бери `createStub`, `createMock` только там, где есть `expects()`.

## Go

```bash
gofmt -l .                # список неотформатированного; исправление: gofmt -w <файлы>
go vet ./...
go build ./...
go test ./...             # для кода с горутинами добавь -race
go mod tidy && go mod verify   # если менялись go.mod, go.sum или импорты; неожиданный дифф go.mod нужно показать
```
Запуск самого сборщика/сервиса может слать сообщения наружу (Telegram и т.п.): без согласия пользователя не запускай.

## Docker Compose

```bash
docker compose config --quiet          # конфиг валиден
docker compose --env-file /dev/null config --quiet   # не зависит от локального .env
docker compose ps
docker compose logs --tail 30 <сервис>
```

## Frontend / Python

Берём из `package.json` (`lint`, `test`, `build`, `typecheck`) и конфигов: `eslint`, `prettier --check`, `tsc --noEmit`; для Python `ruff check`, `ruff format --check` или `black --check`, `mypy`, `pytest`.

## Секреты и мусор в диффе

Файлы, которых не должно быть в индексе:
```
.env.local  .env.*.local  docker/secrets/*  *.pem  *.key  id_rsa*  .DS_Store  .idea/  vendor/  node_modules/  var/  *.cache  *.sqlite  *.dump
```
Проверка индекса:
```bash
git diff --cached --name-only | grep -E '(^|/)(\.env\.local|\.env\..*\.local|docker/secrets/|\.DS_Store|\.idea/|vendor/|node_modules/|var/)|\.(pem|key|dump)$'
```
Поиск значений в добавленных строках (в отчёте значения не печатай, только файл и ключ):
```bash
git diff --cached -U0 | grep -E '^\+' | grep -E '(PASSWORD|SECRET|TOKEN|API_KEY|PRIVATE_KEY)[A-Z_]*\s*[=:]\s*[^\s]{6,}|[0-9]{8,10}:[A-Za-z0-9_-]{30,}|AKIA[0-9A-Z]{16}|-----BEGIN [A-Z ]*PRIVATE KEY-----'
```
Тестовые заглушки (`.env.test`, `change_me`) — не находка, но упомяни.

## Smoke-проверка приложения

```bash
curl -s -o /dev/null -w "%{http_code}\n" -m 15 http://127.0.0.1:<порт>/
curl -s -o /dev/null -w "%{http_code}\n" -m 15 http://127.0.0.1:<порт>/<вход или изменённая страница>
docker compose logs --tail 30 <сервис>        # в свежих логах нет ERROR и исключений
```
Для Symfony полезно заглянуть в `var/log/dev.log`. Админка закрыта логином: код 200 на странице входа и 302 на закрытой странице — норма. Для проверки закрытых страниц под пользователем используй `KernelBrowser::loginUser` в окружении `test` на копии базы (`dbname_suffix: _test`), а не рабочую базу.

Проверка новой миграции без риска для рабочей базы:
```bash
docker compose exec postgres sh -c 'createdb -U $(cat /run/secrets/POSTGRES_USER) -T <db> <db>_test'   # или pg_dump | psql
# прогнать миграцию и тесты в APP_ENV=test, затем удалить копию
docker compose exec postgres sh -c 'dropdb -U $(cat /run/secrets/POSTGRES_USER) <db>_test'
```
Не удаляй чужие базы: имя копии должно быть только твоим.

## Ловушки

- Форматтер находит стилистические правки даже в только что написанном коде (например, php-cs-fixer требует Yoda-условий `null === $x`). Это нормально: применяй его и включай изменённые файлы в коммит.
- Свежий стек (пустая база, `doctrine:schema:validate` красная) — не вина кода: миграции не применены. Скажи об этом и спроси, применять ли.
- Контейнер смонтирован на другой каталог (копия проекта): проверки зелёные, а твой код не проверен. Сверяй `docker inspect` (см. Шаг 0).
- `docker compose exec` не проходит через `entrypoint` контейнера: переменные, которые скрипт запуска экспортирует из файлов секретов, в такой команде могут отсутствовать.
- Запущенный `docker compose watch` держит старый конфиг в памяти и пересоздаёт контейнеры по нему, игнорируя правки `docker-compose.yml`. Если проверки ведут себя странно после правки compose, проверь, не запущен ли `watch`.
- BSD `sed` на macOS: `sed -i ''`, а не `sed -i`.
- `grep` по списку, который не нашёл ничего, возвращает код 1: не принимай это за падение проверки.
