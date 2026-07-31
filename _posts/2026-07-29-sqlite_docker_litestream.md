---
layout: post
title: 'SQLite в Docker: жить хорошо, а хорошо жить — ещё лучше (Litestream)'
date: '2026-07-29 13:44:57 +0000'
tags:
- other
canonical_url: https://blog.rnds.pro/070-litestream?utm_source=teletype&utm_medium=feed_rss&utm_campaign=rnds
original_author: kinnalru

---

![](https://img3.teletype.in/files/26/fa/26fa30c2-bbc2-4f91-9609-899d9f07a9b2.png)

Иногда хочется, чтобы всё было просто. Взял Rails-приложение, завернул в Docker, запустил — и оно работает. Без отдельной СУБД, без монтирования хранилища для картинок и пр.

Но вот незадача: если взять `SQLite`, положить её в контейнер и запустить — при `docker rm` ваша база улетает в никуда. Навсегда. Вместе с пользователями, настройками и всем, что вы так старательно назаполняли.

Сегодня расскажу, как я в pet-проекте [RuNiT](https://runit.rnds.pro/) решил эту проблему с помощью **Litestream** — и просто кайфую.

### Проблема

Классический путь деплоя pet-проекта выглядит так:

1. Поднять `PostgreSQL` в отдельном контейнере (найти под него постоянное хранилище и сервак)
2. Прокинуть volume
3. Понять, что вы тратите больше времени на администрирование СУБД, и хранилище картинок, чем на разработку фич

Для pet-проекта это ощутимый оверхед. `SQLite` решает первый пункт, но создаёт два других:

- файл базы живёт внутри контейнера и при пересоздании исчезает
- бэкапов нет, и если сервер упал — всё, приплыли.

Тут бы и сказать "ну, SQLite не для продакшена", но мне как-то попадался проект [Litestream - Streaming SQLite Replication](https://litestream.io/)

### Litestream

`Litestream` — это standalone-утилита (написана на Go, ~14k stars на [GitHub](https://github.com/benbjohnson/litestream)) для потоковой репликации `SQLite`. Она следит за WAL (Write-Ahead Log) вашей базы и в реальном времени отправляет изменения в S3-совместимое хранилище: `Yandex Object Storage`, `MinIO`, `RustFS` и многие другие.

Умеет:

- **Continuous backup** — каждая транзакция стримится в облако почти мгновенно
- **Restore при старте** — при пересоздании контейнера база автоматически восстанавливается из реплики
- **Point-in-time recovery** — можно откатиться к состоянию на конкретную секунду.
- **Ноль изменений в коде приложения** — `Litestream` работает как отдельный процесс.
- **Дёшево** — object storage стоит копейки.

Звучит как магия? По сути — да. Но магия очень простая и надёжная - разработчик говорит что работает в проде стабильно.

### Как запустить

Тыкать надо тут следующее:

Устанавливаем `Litestream` прямо в образ:

```docker
#Dockerfile

RUN set -ex
  && curl -sL https://github.com/benbjohnson/litestream/releases/download/v0.5.9/litestream-0.5.9-linux-x86_64.deb -o litestream.deb \
  && dpkg -i litestream.deb \
  && rm litestream.deb
```

Передаём credentials для S3 и URL реплики:

```yaml
services:
  app:
    environment:
      - LITESTREAM_ACCESS_KEY_ID=${LITESTREAM_ACCESS_KEY_ID-key}
      - LITESTREAM_SECRET_ACCESS_KEY=${LITESTREAM_SECRET_ACCESS_KEY-secret}
      - REMOTE_S3_FILE=${REMOTE_S3_FILE-s3://11.11.11.11:9000/}
```

Главный трюк — в `docker/start_service.sh`, который является CMD в Dockerfile:

```shell
set -ex

# 1. Восстановить базу из S3, если она не существует локально
litestream restore
  -o storage/${RAILS_ENV}.sqlite3
  -if-replica-exists
  -if-db-not-exists
  ${REMOTE_S3_FILE}/${RAILS_ENV}.sqlite3

# 2. Стандартные Rails шаги (idempotent — безопасны для существующей базы)
bundle exec rails db:create
bundle exec rails db:migrate
bundle exec rails db:seed

# 3. Запускаем Rails ПОД управлением Litestream
exec bundle exec litestream replicate
  -exec "rails s"
  storage/${RAILS_ENV}.sqlite3
  ${REMOTE_S3_FILE}/${RAILS_ENV}.sqlite3
```

Что здесь происходит:

1. **litestream restore** — если локального файла `storage/production.sqlite3` нет, а в S3 есть бэкап, база скачивается. Флаги `-if-replica-exists` и `-if-db-not-exists` делают команду безопасной: она не упадёт, если бэкапа ещё нет (первый деплой), и не перезапишет существующую базу если она есть

2. **db:create db:migrate db:seed** — классика Rails

3. **litestream replicate -exec "rails s"** — `Litestream` запускается как supervisor. Он начинает реплицировать WAL в S3 и одновременно запускает дочерний процесс `rails s`. Если Rails упадёт, `Litestream` тоже завершится.

Результат: контейнер можно остановить, удалить, пересоздать на другом хосте — база восстановится из S3 и продолжит реплицироваться. Красота!

### Профит и подводные камни

Плюсы:

- Минимальная инфраструктура — один контейнер, без отдельной СУБД
- Персистентность — данные не пропадают при удалении контейнера
- Бэкап "из коробки" — каждая транзакция летит в S3, RPO стремится к нулю
- Простое восстановление — `litestream restore `или автовосстановление при старте
- Дёшево — `SQLite` бесплатна, object storage стоит копейки
- Не требует изменений в коде приложения
- Point-in-time recovery — можно восстановить базу на конкретный момент времени

Минусы:

- Только один writer — `SQLite` поддерживает только одного писателя. Это значит, что приложение должно работать в единственном экземпляре (single-node). Ни о каком горизонтальном масштабировании речи не идёт, но в списке ссылок будет сюрприз :)
- Зависимость от object storage — если S3 недоступен, `Litestream` накапливает WAL локально, но при длительном отключении может потребоваться внимание
- Не замена PostgreSQL для высоконагруженных систем — для pet-проекта и небольших сервисов подходит идеально, для HA — нет, но можно посмотреть в ссылочки в конце статьи...

`SQLite` + `Litestream` — это мощная комбинация для «лёгкого» деплоя. Вы получаете полноценную реляционную базу данных в одном файле, непрерывный backup в S3 и автоматическое восстановление при перезапуске контейнера — без необходимости поднимать и обслуживать отдельную СУБД.

Для pet-проекта это часто идеальный баланс между простотой и надёжностью.

Пользуйтесь на здоровье! 🤝

### Мои любимые ссылки, которые стоит почитать:

- [Litestream](https://litestream.io/) и [GitHub](https://github.com/benbjohnson/litestream) — официальный сайт и исходники
- Вот он сюрприз: [LiteFS - Distributed SQLite](https://fly.io/docs/litefs/) - как сделать, горизонтальное масштабирование с `SQLite`. Говорят, работает в проде :)
- Ну и ссылочка на сам проект [Беговой Клуб -=БеГиМ=-](https://runit.rnds.pro/)

* * *
<tt-tags id="DhCr">
    <tt-tag name="sqlite">#sqlite</tt-tag>
    <tt-tag name="rails">#rails</tt-tag>
    <tt-tag name="database">#database</tt-tag>
    <tt-tag name="replicantion">#replicantion</tt-tag>
  </tt-tags>