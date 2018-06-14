---
layout: post
title: "Настройка собственного rubygems сервера"
date: 2018-06-14 17:00:00 +0300
tags: ruby, rubygems, nexus
---

Иногда компании хочется обезопасить себя от возможных перебоев в работе https://rubygems.org. Для этого можно запустить кэширующих прокси, который будет пропускать через себя все запросы к rubygems.org и кэшировать используемые гемы у себя.

В этой заметке я расскажу об особенностях использования репозитория артефактов [Nexus](https://www.sonatype.com/nexus-repository-oss).  Я не буду вдаваться в подробности установки Nexus (это подробно описано в [документации](BUNDLE_MIRROR__RUBYGEMS__ORG)) и сконцентрируюсь на том, как его использовать.

Для рубиста Nexus предоставляет два типа репозиториев rubygems (proxy) и rubygems (hosted). Первый тип предназначен для проксирования доступа к rubygems.org, а второй тип позволяет хранить собственные не публичные гемы. 

Чтобы создать репозиторий, нужно в панели управления Nexus выбрать Repositories -> Create Repository -> rubygems (proxy) или rubygems (hosted).  После этого в списке в списке репозиториев, можно скопировать ссылку на репозиторий. 

![Список репозиториев]({{ "/assets/nexus/list.png" | absolute_url }})

Чтобы воспользоваться ими,  в файле `Gemifle` нужно добавить два источника гемов:

```ruby
source 'https://{login}:{password}@nexus.example.com/repository/gems-public/'
source 'https://{login}:{password}@nexus.example.com/repository/gems-private/'

# ...
```

Теперь каждый раз выполняя `bundle install`, гемы будут кэшироваться на нашем прокси.

Nexus имеет ещё один тип репозиториев — rubygems (group). Этот тип репозитория позволяет сгруппировать несколько других репозиториев так, чтобы пользователи получали к ним доступ по одной ссылке. 

Например, можно сгруппировать ранее созданные репозитории `gems-public` и `gems-private` в группу `gems`, и тогда в нашем гемфайле будет только один источник, дающий доступ как к приватным гемам, так и к проксируемым:

```ruby
source 'https://{login}:{password}@nexus.example.com/repository/gems/'

# ...
```

Уже лучше. Но что, если наш код лежит в публичном месте (например, на гитхабе), а при разработке мы хотим использовать наш прокси? [Bundler](http://bundler.io) имеет средства для решения этой проблемы. Он позволяет [сконфигурировать “зеркало”](https://bundler.io/v1.5/bundle_config.html#gem-source-mirrors-1) для любого источника. 

Например, чтобы использовать наш прокси в качестве зеркала:

```
> bundle config mirror.https://rubygems.org https://nexus.example.com/repository/gems/
> bundle config nexus.example.com {login}:{password}
```

Теперь можно вернуть Gemfile к первоначальному виду:

```ruby 
source "https://rubygems.org"
```

Так как конфигурация `bundle config` записывается в файле `.bundle/config` который не хранится в репозитории, информация о том, что мы используем прокси (и логин/пароль к нему) известна только нам. 

Что если мы хотим использовать прокси на CI сервере (gitlab-ci, travis) но при этом не хранить логин и пароль от нашего прокси в репозитории? У Bundler есть решение и этой проблемы. 

[Bundler позволяет](https://bundler.io/v1.16/bundle_config.html#LIST-OF-AVAILABLE-KEYS) конфигурировать себя как при помощи `bundle config`, так и при помощи переменных окружения. Например, чтобы сконфигурировать наш прокси, нужно:

```bash
> export BUNDLE_MIRROR__RUBYGEMS__ORG=https://nexus.example.com/repository/gems/
> export BUNDLE_NEXUS__EXAMPLE__COM={login}:{password}
> bundle install
```

Большинство CI систем позволяют хранить переменные окружения, скрывая их от случайных наблюдателей, поэтому из не придётся
экспортировать и они не будут видны в логе билда.