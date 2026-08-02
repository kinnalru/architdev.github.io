---
canonical_url: https://blog.rnds.pro/064-devsecops1?utm_source=teletype&utm_medium=feed_rss&utm_campaign=rnds
cover_image: https://img1.teletype.in/files/42/f0/42f0e9b3-d3f8-403f-ac95-bfd789c92278.jpeg
date: '2025-02-11 12:34:47 +0000'
image: https://img1.teletype.in/files/42/f0/42f0e9b3-d3f8-403f-ac95-bfd789c92278.jpeg
layout: post
original_author: kinnalru
published: 'true'
published_at: '2025-02-11 12:34:47 +0000'
slug: devsecops_podkralsya_nezametno_hotya_zameten_byl_izdaleka
tags: []
title: DevSecOps подкрался незаметно, хотя заметен был издалека…
---
![](https://img1.teletype.in/files/42/f0/42f0e9b3-d3f8-403f-ac95-bfd789c92278.jpeg)

DevSecOps уверенно шагает по нашей индустрии, и горе тому, кто попадёт под его поступь… Эта статья про ~~ультимативную~~ сборку базовых образов для Ruby для удовлетворения самых параноидальных потребностей ИБ. Да, именно об этом мы и расскажем - что такое "инсталляция ruby", где, что, почему лежит и как с этим жить нашему пайплайну сборки и самому приложению.

<section style="background-color:hsl(hsl(24, 24%, var(--autocolor-background-lightness, 95%)), 85%, 85%);">
    <p id="CaKt">Статья подойдёт тем, кто хочет более глубоко понимать, какие процессы происходят в системе, когда вызывается <code>gem install</code>, <code>bundle install</code> или (не дай Бог) <code>gem update –system</code></p>
  </section>
## Как было до?

С самого начала появления docker мы в RNDSOFT использовали парадигму "всё включено". Для нас это означало, что в образ мы включаем всё, что нам надо не только для продуктовой эксплуатации, но и для проведения всех тестов. При прохождении CI пайплайна на следующие стадии продвигался образ целиком и, в конце концов, выкатывался на прод.

Это было очень удобно и позволяло быть максимально (насколько это вообще возможно) уверенным в работоспособности, однако имело ряд серьёзных минусов, с которыми мы прекрасно жили достаточно долгое время:

- размер образа был большим - от 1 ГБ;
- образ включал большое количество неиспользуемого (в проде) ПО.

## Сканеры сканировали, сканировали, да не высканировали…

И вот однажды один (на данный момент уже далеко не один) наш клиент захотел устранения всех замечаний, которые смог выявить сканнер [Trivy](https://github.com/aquasecurity/trivy). А их, как не трудно догадаться, было достаточно много. И главная проблема заключается в том, что большая часть замечаний никак не связана с непосредственными зависимостями вашего приложения (теми, которые фиксируются в Gemfile.lock).   
Так откуда же они берутся?

Если коротко отвечать - отовсюду :) При сборке проекта необходимо поставить его зависимости - это план минимум. Кроме того, часто появляется необходимость установить bundle определенной версии или даже обновить ruby целиком, выполнив команду [gem update --system](https://guides.rubygems.org/command-reference/#gem-update). В результате этих действий в ваш образ в разнообразные папки ставятся разнообразные гемы, но

<section style="background-color:hsl(hsl(323, 50%, var(--autocolor-background-lightness, 95%)), 85%, 85%);">
    <p id="gt1O">старые версии гемов и сама базовая "инсталляция ruby" остаётся в системе со всеми своими "устаревшими" и "уязвимыми" версиями. И эти версии очень нравятся сканерам для того, чтобы поднять тревогу. </p>
  </section>

И еще два слова о том, что же именно сканирует Trivy (касательно ruby конечно):

- сканирует [.gemspec файлы](https://guides.rubygems.org/specification-reference/) на всей файловой системе и не важно, установлен гем или нет - если в .gemspec указана версия "с уязвимостью" - алярм;
- сканирует .gem файлы на всей файловой системе и не важно, установлен гем, лежит просто в папке cache, используется или нет в вашем приложении.

Значит для удовлетворения хотелок сканера, надо сделать так, чтоб нигде не лежало ничего лишнего или не используемого, **включая** [stdlib](https://docs.ruby-lang.org/en/3.2/standard_library_rdoc.html) - стандартную библиотеку ruby.

Проблема ясна, задача очевидна - можно приступать к решению и начинать надо с базовых образов. Их надо собрать так, чтобы сами базовые образы уже не содержали никаких уязвимостей.

![](https://img3.teletype.in/files/e1/7a/e17a3417-a64b-4027-adf2-971f332cf9dd.png)
_Немного удручающее зрелище, особенно для ИБ клиента_

## Что такое инсталляция ruby?

Если мы возьмём любой дистрибутив с установленным там ruby, то увидим, что само ruby будет находиться где-то в районе:

- /usr/lib/ruby - например в alpine после `apk add ruby`
- /usr/local/lib/ruby - например в `ruby:alpine`, где руби собирается отдельно от apk

А дальше начинается интересное. Если посмотреть в финальный образ, в котором сделано много различных операций (обновление ruby, установка гемов через `gem install`, установка через `bundle install`), то в корневой папке ruby обнаружится несколько папок, по которым тем или иным способом будут распределены установленные вами гемы:

```plaintext
/usr/local/lib/ruby
├── 3.2.0
├── gems/3.2.0
├── site_ruby/3.2.0
└── vendor_ruby/3.2.0
```

Сразу добавим к этому списку другие папки (которые можно посмотреть в выводе команды `gem env`):

```yaml
 - INSTALLATION DIRECTORY: /usr/local/bundle
 - USER INSTALLATION DIRECTORY: /root/.local/share/gem/ruby/3.3.0
 - SPEC CACHE DIRECTORY: /root/.cache/gem/specs
 - GEM PATHS:
    - /usr/lib/ruby/gems/3.3.0
    - /root/.local/share/gem/ruby/3.3.0
```

И вишенкой на торте будет настройка вашего bundle, если вы используете кеширование сборки ([bundle cache](https://bundler.io/man/bundle-cache.1.html) или `bundle config set cache_path vendor/cache`), и используемые в этом зоопарке переменные окружения `GEM_HOME`, `BUNDLE_CACHE_PATH`, `BUNDLE_PATH` и скорее всего еще какие-то скрытые в глубинах экосистемы ruby.

Немного путано и в результате беспорядочных ~~связей~~ установок и обновлений во всех этих папках могут (и будут) появляться гемы. Надо исправить!

Постараюсь дать верхнеуровневое описание, что же именно это за папочки, не углубляясь в подробности и исключения:

### /usr/local/lib/ruby/3.3.0

Именно в этой папке установлены основные файлы ruby и stdlib, а также компилируемые расширения в папке x86\_64-linux-musl (в нашем случае собранные alpine для x86\_64). Эти файлы принадлежат условной "системе" и никак не будут изменяться при дальнейших модификациях, например при установке или обновлении гемов через `gem install`. Вместо этого библиотеки будут ставиться в папку `/usr/local/lib/ruby/gems/3.3.0`

### /usr/local/lib/ruby/gems/3.3.0

Тут собрано всё, что вы ставите (без модификации `GEM_PATH`) командами `gem install,` включая компилируемые расширения.

### /usr/local/lib/ruby/vendor\_ruby/3.3.0

Сюда **должны** ставиться дополнительные гемы и/или патчи от команды мейнтейнеров дистрибутива. Об этом сложно найти информацию, но несколько слов есть в книге `The Ruby Programming Language` или, что интереснее, в changelog для [NEWS for Ruby 1.8.7](https://docs.ruby-lang.org/en/2.3.0/NEWS-1_8_7.html) (да, окаменелое…):

<section style="background-color:hsl(hsl(24, 24%, var(--autocolor-background-lightness, 95%)), 85%, 85%);">
    <p id="VAZR"><strong>vendor_ruby</strong> directory<br>A new library directory named <code>vendor_ruby</code> is introduced in addition to <code>site_ruby</code>. The idea is to separate libraries installed by the package system (<code>vendor</code>) from manually (<code>site</code>) installed libraries preventing the former from getting overwritten by the latter, while preserving the user option to override vendor libraries with site libraries. (<code>site_ruby</code> takes precedence over <code>vendor_ruby</code>)<br>If you are a package maintainer, make each library package configure the library passing the <code>--vendor</code> option to <code>extconf.rb</code> so that the library files will get installed under <code>vendor_ruby</code>.<br>You can change the directory locations using configure options such as<code> --with-sitedir=DIR</code> and <code>--with-vendordir=DIR</code>.</p>
  </section>
### /usr/local/lib/ruby/site\_ruby/3.3.0

Сюда будут ставиться "системные" файлы, но не от OS, а от самого ruby, например после выполнения `gem update –system`:

```plaintext
site_ruby/
└── 3.4.0
   ├── bundler
   ├── rubygems
   └── x86_64-linux-musl
```

### /usr/local/lib/ruby/gems/3.3.0/specifications/

…а также INSTALLATION DIRECTORY `/usr/local/bundle/specifications`…  
…а также USER INSTALLATION DIRECTORY: `/root/.local/share/gem/ruby/3.3.0`…  
…а также SPEC CACHE DIRECTORY `/root/.cache/gem/specs`…  
…а также BUNDLE\_CACHE\_PATH …

Сюда ставятся .gemspec файлы, которые собственно и говорят пакетному менеджеру ruby (gem и bundle), какие именно версии каких гемов установлены.

### /usr/local/lib/ruby/gems/3.3.0/specifications/default

Default, Карл… Default - это особое состояние гема, и эти гемы нельзя удалить. Если вы обновили гем до новой версии, то старая всё равно останется, и Trivy вам этого не простит. Весьма вредная папка с точки зрения сканирования.

### /usr/local/lib/ruby/gems/3.3.0/cache/

…а также INSTALLATION DIRECTORY `/usr/local/bundle/cache`…  
…а также BUNDLE\_PATH …

Сюда пакетный менеджер ruby (gem или bundle) скачивает гемы (допустим вы ставите faraday.gem) перед установкой. Этот кеш очень часто используется для ускорения сборки, например [в вашем Gitlab](https://docs.gitlab.com/ee/ci/caching/#cache-ruby-dependencies).

## Что здесь у вас происходит?!

После того как мы в процессе исследования детально разобрались и увидели всё это многообразие мест, где могут находиться файлы, вызывающие панические атаки у Trivy, мы решили радикально решить эту проблему: свести все файлы, все гемы и все спеки (.gemspec) в одно место. Это позволит легко следить за всеми (всеми!) фактическими зависимостями и эффективно пользоваться командами `gem cleanup` и `bundle clean`. Тут надо отметить, что для вашей рабочей OS (системы общего назначения) такое решение приведёт к поломке системного пакетного менеджера ([Gentoo Portage](https://ru.wikipedia.org/wiki/Portage), [dpkg/apt](https://ru.wikipedia.org/wiki/Dpkg), [rpm/zypper](https://ru.wikipedia.org/wiki/RPM) и пр.), но мы ведь говорим о конкретной сборке ruby под ваш конкретный проект - и тут никаких проблем не будет.

<section style="background-color:hsl(hsl(24, 24%, var(--autocolor-background-lightness, 95%)), 85%, 85%);">
    <p id="ma0i">Для того чтобы узнать что куда, когда и зачем ставится, мы использовали git прямо на корне файловой системы внутри контейнера (ruby:alpine, просто alpine, ruby:debian и другие образы для сравнения) и фиксировали изменения после различных команд ⏳</p>
  </section>

Но это еще не всё. Остаётся еще проблема с default gemspec, но её относительно легко решить:

```shell
mv /usr/local/lib/ruby/gems/3.3.0/specifications/default/* /usr/local/lib/ruby/gems/3.3.0/specifications/
```

Теперь мы можем сформулировать План:

- Все дороги ведут в Рим - делаем ссылочки для `vendor_ruby`, `site_ruby` и пр. Также явно прописываем системные переменные `GEM_HOME` и `BUNDLE_APP_CONFIG.`
- Разбираемся с default gems.
- Обновляем ruby (имеется в виду stdlib) до последней требуемой версии.
- Обновляем bundle до нужной версии.
- Удаляем лишние гемы (например rdoc).
- Переставляем (!) системные (на текущий момент сборки базового образа - **все** ) гемы, потому что, как оказалось, `mv` для default gemspec имеет не очень хорошие последствия.
- profit!

Сказано - сделано, и добро пожаловать под кат!

```docker
ARG BASE_RUBY=3.2
ARG BASE_ALPINE=alpine3.16
ARG BASE_IMAGE=ruby:${BASE_RUBY}-${BASE_ALPINE}

FROM ${BASE_IMAGE}

ARG BASE_RUBY=3.2

ARG RUBYGEMS_VERSION=3.5.20
ARG BUNDLER_VERSION=2.5.20

# эта переменная есть в старом alpine но нет в debian и новом
# добавляем потому что она очень нужна для работы с папочками
ENV RUBY_MAJOR=${BASE_RUBY}

ENV RUBYGEMS_VERSION=${RUBYGEMS_VERSION} \
    BUNDLER_VERSION=${BUNDLER_VERSION}

RUN apk update && apk upgrade

# Это наш костыльный скрипт который удаляет всякие лишние кеши,
# man-файлы и прочий мусор. 
COPY common/scripts/cleanallbuilds.sh /usr/bin/

# dumb-init всегда используем как PID-1 но это немного другая история
RUN set -ex \
 && apk add dumb-init \
 && cleanallbuilds.sh

###### 
# Надругиваемся над диструбутивом, чтоб иметь строго
# одну версию руби в систему и управлять ею целиком через gem/bundle

# все пути ведут в Рим
ENV GEM_HOME=/usr/local/lib/ruby/gems/${RUBY_MAJOR}.0/
ENV BUNDLE_APP_CONFIG=/usr/local/lib/ruby/gems/${RUBY_MAJOR}.0/

# Сводим vendor_ruby, site_ruby и GEM_HOME в одно место
RUN set -ex \
 && rm -rf /usr/local/lib/ruby/site_ruby /usr/local/lib/ruby/vendor_ruby \
 && ln -sf /usr/local/lib/ruby /usr/local/lib/ruby/site_ruby \
 && ln -sf /usr/local/lib/ruby /usr/local/lib/ruby/vendor_ruby \
 && mkdir -p /root/.local/share/gem/ruby/ \
 && ln -sf ${GEM_HOME} /root/.local/share/gem/ruby/${RUBY_MAJOR}.0

# Чутка тюним bundle config чтоб в дальнейшем не забыть
RUN set -ex \
 && bundle config --local disable_version_check true \
 && bundle config --local clean false \
 && bundle config --local no_prune false \
 && bundle config --local disable_local_branch_check true \
 && bundle config --local jobs 2 \
 && bundle config --local allow_offline_install true

# настраиваем .gemrc чтоб не было ничего лишнего, вклюячая rdoc
RUN set -ex \
 && echo 'gem: --no-document' > /usr/local/etc/gemrc \
 && echo 'update_sources: false' >> /usr/local/etc/gemrc \
 && echo 'verbose: false' >> /usr/local/etc/gemrc \
 && echo 'update: --no-suggestions' >> /usr/local/etc/gemrc \
 && echo 'install: --no-suggestions --conservative' >> /usr/local/etc/gemrc

# пытаемся удалить ненужные гемы с самого начала - попытка не пытка
RUN set -ex \
 && gem uninstall -a -x --quiet --force `gem list | cut -f 1 -d " "` \
 && gem cleanup

RUN set -ex \
 && apk add --virtual .build-deps \
   autoconf \
   bison \
   bzip2 \
   bzip2-dev \
   coreutils \
   curl-dev \
   dpkg-dev dpkg \
   g++ \
   gcc \
   gdbm-dev \
   git \
   glib-dev \
   libc-dev \
   libffi-dev \
   libxml2-dev \
   libxslt-dev \
   linux-headers \
   make \
   ncurses-dev \
   procps \
   readline-dev \
   tar \
   xz \
   yaml-dev \
   zlib-dev \
   shared-mime-info \
 && mv /usr/local/lib/ruby/gems/${RUBY_MAJOR}.0/specifications/default/* /usr/local/lib/ruby/gems/${RUBY_MAJOR}.0/specifications/ \
 && gem cleanup \
 && gem update --system "${RUBYGEMS_VERSION}" \
 && gem uninstall bundler --all --silent || true \
 && gem install bundler -v "${BUNDLER_VERSION}" \
 && gem uninstall rdoc --all --silent || true \
 && GMS=`gem list | sed s/default:\ // | sed -E 's/\ \((.*)\)/:\1/' | sort` \
 && DEBUG_FLAGS="-Wno-calloc-transposed-args" gem pristine --all --extensions \
 && commands=$(for x in $GMS; do \
   g=$(echo "$x" | cut -f 1 -d ":"); \
   v=$(echo "$x" | cut -f 2 -d ":"); \
   echo gem pristine $g --version "$v"; \
 done) \
 && echo -e "$commands" | xargs -I CMD -P 3 bash -c CMD\
 && gem cleanup \
 && rm -rf root/.local/share/gem/specs \
 && apk del .build-deps \
 && cleanallbuilds.sh

WORKDIR /home/app
RUN set -ex \
 && adduser -D -s /sbin/nologin app \
 && chown -R app:app /home/app

ENTRYPOINT ["/usr/bin/entrypoint.sh"]
SHELL ["/bin/sh", "-c"] 
```

Самое интересное, конечно, находится в одном слое самого пухленького RUN, и по некоторым командам надо дать пояснения:

- `GMS=`gem list | sed s/default:\ // | sed -E 's/\ \((.*)\)/:\1/' | sort`` - получаем список установленных гемов с версиями в формате "yaml:0.4.0 zlib:3.2.1". Это нам потребуется дальше из-за странного поведения `gem pristine`
- `gem pristine --all --extensions` - должен для всех установленных гемов сделать "чистовую" установку, включая скачивание и перекомпиляцию расширений. Но нет. На самом деле он делает не для всех и не всё :(
- `DEBUG_FLAGS="-Wno-calloc-transposed-args"` - мы используем один Dockerfile для сборки базовых образов ruby 2.7, 3.0, 3.1, 3.2, и по умолчанию все расширения должны компилироваться без единого ~~разрыва~~ ворнинга компиляции, но для некоторых версий руби (кажется 3.0) это не так, и именно этот ворнинг всё портит, поэтому его подавляем без затей.
- `commands=$(for x in $GMS; do …` - а вы знали, что в bash тоже есть пул потоков? Строго говоря, он [пул процессов](https://stackoverflow.com/questions/6441509/how-to-write-a-process-pool-bash-shell), и не в баше, а в xargs, но это не важно. В общем тут список `$GMS` в виде "yaml:0.4.0 zlib:3.2.1" трансформируется в список `$commands`, потому что в такой форме `gem pristine` работает именно так как надо:

```shell
gem pristine yaml --version "0.4.0"
gem pristine zlib --version "3.2.1"
...
```

- `echo -e "$commands" | xargs -I CMD -P 3 bash -c CMD` - отправляем `$commands` в пул из трех процессов для одновременной установки гемов. 
- `cleanallbuilds.sh` - зовём в самом конце кустарный скрипт, который удаляет всякий мусор из системы:

```shell
#!/bin/sh

# ruby (alpine + debian)
# Скрипт единый идля debian и для alpine, поэтому ошибки,
# возникающие из-за разницы дистрибутивов просто игнорируем
rm -rf /usr/src/ruby/ || true
rm -rf /root/.local/share/gem/specs/ || true
rm -rf /root/.local/state/ || true
rm -rf /root/.cache/gem/ || true
rm -rf /usr/local/lib/ruby/gems/3.2.0/cache/ || true
rm -rf /root/.bundle/cache/ || true
gem cleanup || true

# alpine
apk cache clean || true

# debian
apt-get clean autoclean || true
apt-get autoclean --yes || true
apt-get autoremove --yes || true

find /var/log/ -type f -delete || true
find /var/lib/log/ -type f -delete || true
find /usr/share/doc/ -type f -delete || true
```

## Результат

Старый образ занимал 900MB, новый - 152MB (не спрашивайте...)

![](https://img1.teletype.in/files/c3/95/c395d60f-18a0-45a0-9757-f7b7f779949c.png)
_Значительно лучше. Особенно для базового образа_

## Что дальше?

Теперь эти базовые образы надо начать использовать непосредственно в CI пайплайнах боевых сервисов и, конечно, процедуру сборки и тестирования придётся поменять. Точнее, мы уже перешли на новые образы и сборку уже пару недель назад - дело теперь только за статьёй. А в качестве приманки приведу результаты перехода для одного из самых "толстых" наших сервисов:

```plaintext
image:old size:1.49GB vulns:255
image:new-prod size:259MB vulns:1 (Вчерашняя!! СVE-2025-25186)
image:new-test size:269MB vulns:1 (СVE-2025-25186)
```

Если взглянуть ретроспективно, то совсем не весело - так что:

<section style="background-color:hsl(hsl(24, 24%, var(--autocolor-background-lightness, 95%)), 85%, 85%);">
    <p id="tf2v">профилируйте чаще не только ваш код и инфраструктуру сборки.</p>
  </section><tt-tags id="HMo0">
    <tt-tag name="docker">#docker</tt-tag>
    <tt-tag name="devsecops">#devsecops</tt-tag>
    <tt-tag name="ruby">#ruby</tt-tag>
    <tt-tag name="cicd">#cicd</tt-tag>
  </tt-tags>