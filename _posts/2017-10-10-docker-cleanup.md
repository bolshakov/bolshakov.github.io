---
layout: post
title: "Шпаргалка по очистке docker контейнеров"
date: 2017-10-10 17:00:00 +0300
tags: ruby
---

Остановить все запущенный контейнеры:

```
docker ps --quiet --filter status=running | xargs --no-run-if-empty docker stop
```

Удалить все остановленные контейнеры:

```
docker ps -a --filter status=exited --quiet | xargs --no-run-if-empty docker rm
```

Удалить все data-контейнеры:  

```
docker volume ls --quiet | xargs --no-run-if-empty docker volume rm
```

Ну и всё одной командой:

```
docker ps --quiet --filter status=running | xargs --no-run-if-empty docker stop; docker ps -a --filter status=exited --quiet | xargs --no-run-if-empty docker rm; docker volume ls --quiet | xargs --no-run-if-empty docker volume rm
``` 

*UP*: Есть более простой способ сделать всё тоже самое:

```
docker system prune
```
