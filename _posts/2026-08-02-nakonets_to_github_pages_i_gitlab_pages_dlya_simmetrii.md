---
canonical_url: https://teletype.in/@jerry_ru/x7lDs3PQUub?utm_source=teletype&utm_medium=feed_rss&utm_campaign=jerry_ru
cover_image: https://img3.teletype.in/files/22/f5/22f5b394-0142-43f3-96d5-dc955e05ff2f.jpeg
date: '2026-08-02 12:22:16 +0000'
image: https://img3.teletype.in/files/22/f5/22f5b394-0142-43f3-96d5-dc955e05ff2f.jpeg
layout: post
original_author: kinnalru
published: 'true'
published_at: '2026-08-02 12:22:16 +0000'
slug: nakonets_to_github_pages_i_gitlab_pages_dlya_simmetrii
tags:
- блог
- synchronization
- jekyll
title: Наконец-то GitHub Pages (и GitLab Pages — для симметрии)
---
![](https://img3.teletype.in/files/22/f5/22f5b394-0142-43f3-96d5-dc955e05ff2f.jpeg)

Давно присматривался к [GitHub Pages](https://pages.github.com/), но руки никак не доходили.

Внезапно решил навести порядок на площадках, где публикую статьи и заметки, и решился-таки на статику (как доп. площадку).

## Всякие разные Pages

`GitHub Pages` нативно поддерживает [Jekyll](https://jekyllrb.com/) — пушнул в репозиторий, и сайт сам собрался. Никаких докеров, никаких CI/CD (ну почти), никаких серверов.

А чтобы было совсем хорошо, решил поддерживать и `GitLab Pages` — для симметрии. Чтоб не привязываться к одной платформе и чтоб было куда переключиться, если что. Но тут без CI/CD никак.

### Структура репозитория и настройки

Репозиторий называется как положено — `kinnalru.github.io` - тогда он будет доступен по этому доменному имени. Gemfile:

```ruby
gem "github-pages", group: :jekyll_plugins

group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-sitemap"
end
```

Обратите внимание на `github-pages` — это мета-гем, который тянет `Jekyll` и все нужные плагины в совместимых версиях. Никаких "а у меня работает", потому что `GitHub Pages` собирает ровно тем же набором.

Заполняем конфигурацию [\_config.yml](https://github.com/kinnalru/kinnalru.github.io/blob/master/_config.yml) где выбираем тему, плагины, и всякие настройки. Тема [minima](https://github.com/jekyll/minima) — адаптивная под светлую/темную тему. И сразу же в конфиге ссылочки на все свои ресурсы: **GitLab** , **GitHub** , **Teletype** , **Telegram** , **Dev.to**. Чтоб гость зашел и сразу понял, где меня искать, а то вдруг нигде раньше не нашел ссылок :)

### GitLab Pages — симметрия ради симметрии

Для `GitLab Pages` пришлось чуть заморочиться. `GitHub Pages` собирает `Jekyll` сам, а `GitLab` — нет, нужен [.gitlab-ci.yml](https://github.com/kinnalru/kinnalru.github.io/blob/master/.gitlab-ci.yml):

```yaml
image: ruby:3.2-slim

build:
  script:
    - apt-get update -y; apt-get install -y build-essential
    - bundle
    - JEKYLL_ENV=production bundle exec jekyll build --config _config.yml,_config.gitlab.yml -d ./public/
  pages:
    publish: public
  rules:
    - if: ($CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH) && ($CI_PIPELINE_SOURCE != "schedule")
```

И отдельный `_config.gitlab.yml` — переопределяем только **url** :

```yaml
url: https://kinnalru.gitlab.io
```

`Jekyll` умеет мержить конфиги при сборке: `--config _config.yml,_config.gitlab.yml`. Всё, что идет справа, переопределяет то, что слева.

### Чтоб было "хорошо" — намазал сверху RSS и Sitemap

Просто собирать HTML — это скучно. Мне нужно, чтоб поисковики индексировали, а читатели подписывались. Поэтому сверху еще и:

- ставим плагин `jekyll-sitemap`, и он автоматически генерирует `/sitemap.xml`. Но я не доверяю автоматике до конца, поэтому в `_includes/custom-head.html` добавил явную ссылку:

```html
  <link rel="sitemap" type="application/xml" title="Sitemap" href="/sitemap.xml" />
```

- ставим плагин `jekyll-feed` — он генерирует базовый `/feed.xml`
- но я решил, что хочу RSS с полным контентом статей и расширенными настройками, поэтому написал свой шаблон [feed.xml](https://github.com/kinnalru/kinnalru.github.io/blob/master/feed.xml)

### Кастомные шаблоны

Стандартная minima хороша, но мне хотелось, чтоб на каждом посте была ссылка на оригинальную публикацию. Переопределил `_layouts/post.html` и `_layouts/home.html` — добавил **canonical\_url** и пометку _"Моя публикация на ..."_. Теперь зеркало не выглядит как украденный контент, а как полноценный агрегатор.

### Что в итоге?

В результате мы имеем:  
- Статический сайт на `Jekyll`, который собирается автоматически  
- Зеркало на `GitHub Pages` — [kinnalru.github.io](https://kinnalru.github.io)  
- Зеркало на `GitLab Pages` — [kinnalru.gitlab.io](https://kinnalru.gitlab.io)  
- `RSS` с полным контентом для подписчиков и дальнейшей обработки  
- `Sitemap` для поисковиков  
- Все ссылки на мои ресурсы в одном месте

> Keep it simple, stupid — но при этом всё должно работать.

* * *

[Teletype](https://teletype.in/@jerry_ru) | [Архи DEV](https://t.me/architdev) | [Dev.to](https://dev.to/kinnalru) | [GitHub](https://github.com/kinnalru) | [Blogger](https://kinnalru.blogspot.com)