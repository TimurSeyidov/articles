# Makefile для быстрого управления

**Makefile** - это файл конфигурации, который используется программой Make. Make - это утилита автоматизации, которая позволяет описывать зависимости между исходными файлами и целями (действиями), которые нужно выполнить. Обычно Make используется для автоматизации процесса компиляции программного обеспечения.

Makefile содержит информацию о том, из каких исходных файлов состоит программа, какие цели нужно выполнить (обычно это компиляции исходных файлов в бинарные файлы), а также описания зависимостей между исходными файлами и целями. Когда Make запускается, он проверяет, все ли цели, указанные в Makefile, выполнены. Если есть невыполненные зависимости, Make выполняет их и переходит к следующей цели. Если все цели выполнены, Make завершает работу.

Создадим файл **Makefile**

```make
init: docker-down-clear docker-pull docker-build docker-up
down: docker-down-clear
restart: down init

docker-up:
    docker-compose up -d

docker-down:
    docker-compose down --remove-orphans

docker-down-clear:
    docker-compose down -v --remove-orphans

docker-pull:
    docker-compose pull

docker-build:
    docker-compose build
```

**docker-up**
Собирает и запускат контейнеры

**docker-down**
Удаляет все контейнеры, образы и сети, определенные в файле docker-compose.yml. Опция **--remove-orphans** отвечает за удаление всех оставшихся ресурсов, которые не были явно удалены в файле docker-compose.yml. 

**docker-down-clear**
Удаляет все контейнеры, образы и сети, определенные в файле docker-compose.yml. Опция **--remove-orphans** отвечает за удаление всех оставшихся ресурсов, которые не были явно удалены в файле docker-compose.yml. Опция **-v** удаляет все контейнеры, сети и тома, определенные в вашем файле docker-compose.yml.

**docker-pull**
Cкачивает образы, указанные в файле docker-compose.yaml

**docker-build**
Создает образы на основе Dockerfile, указанного в файле docker-compose.yaml