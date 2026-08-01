---
title: Опять синхронизация, но теперь блогов
published: 'true'
date: '2026-07-31 13:26:25 +0000'
original_author: kinnalru
tags: []
canonical_url: https://teletype.in/@jerry_ru/scWep5L3rOI?utm_source=teletype&utm_medium=feed_rss&utm_campaign=jerry_ru
slug: opyat_sinhronizatsiya_no_teper_blogov
published_at: '2026-07-31 13:26:25 +0000'
layout: post
---
Есть у меня небольшой фетиш по синхронизации… И в очередной раз, подойдя к проблеме блога, я наконец собрал разрозненные куски в единую систему :)

### Кто к нам с чем и зачем?

За годы у меня накопилась целая коллекция площадок, куда я (и не только я) что-то публикую:

- **Личный блог** на [Teletype](https://teletype.in/@jerry_ru) — сюда пишу руками
- **Корпоративный блог** [blog.rnds.pro](https://blog.rnds.pro/) — пишем вместе с коллегами
- **Dev.to** — личный аккаунт [kinnalru](https://dev.to/kinnalru) и корпоративная организация [RNDSOFT](https://dev.to/rnds)
- **GitHub Pages** — [kinnalru.github.io](https://kinnalru.github.io)
- **Blogger (Blogspot)** — [kinnalru.blogspot.com](https://kinnalru.blogspot.com)

Раньше тут обитал ещё и ~~Hashnode~~ , но теперь у этих негодяев платное API для публикации — пусть сами к себе пишут.

### Как теперь работает схема

1. Пишу статью вручную в Teletype (личное) или в корпоративный блог.
2. Статьи автоматически улетают в черновики `Dev.to` через `RSS`
3. Перед публикацией на Dev.to я добавляю **FrontMatter** `original_author=kinnalru`.
4. Публикуем: личные — от себя, корпоративные — от организации [RNDSOFT](https://dev.to/rnds)
5. По крону (через GitLab Pipelines) запускается скрипт-синхронизатор, который забирает только мои статьи (по фильтру `original_author`) и публикует их дальше: 

  - на GitHub Pages
  - на Blogger

### Итог

Написал небольшой синхронизатор, который делает всё ровно так, как мне нужно. Теперь дело за малым — писать сами статьи :)

* * *

[Teletype](https://teletype.in/@jerry_ru)| [Архи DEV](https://t.me/architdev) | [Dev.to](https://dev.to/kinnalru) | [GitHub](https://kinnalru.github.io) | [Blogger](https://kinnalru.blogspot.com)