# Prometheus

**Prometheus** - это система мониторинга с открытым исходным кодом, разработанная программистами из SoundCloud. Она собирает данные о работе приложений, инфраструктуры и сервисов и сохраняет их для последующего анализа. Prometheus используется для выявления проблем в работе систем, анализа производительности и оптимизации работы приложений.

**Prometheus** использует различные методы для сбора данных, такие как опрос сервисов, чтение метрик из различных источников, таких как **Apache Kafka** или **Graphite**, и использование exporters для сбора данных от различных сервисов и систем. Собранные данные сохраняются в базе данных **Prometheus**, которая может быть визуализирована с помощью [Grafana](../grafana/README.md) или других инструментов визуализации.

**Prometheus** также поддерживает механизм оповещений, который позволяет автоматически отправлять уведомления при обнаружении определенных проблем или достижении пороговых значений метрик. Это позволяет быстро реагировать на возникающие проблемы и предотвращать их влияние на работу сервисов и приложений.

Для начала создадим в корне проекта **Makefile** ([подробнее тут](../makefile/README.md))

Теперь выполним предварительные действия.

## Настройка брандмауэра
Если на вашем сервере работает брандмауэр, необходимо открыть порт:
- **9090/tcp** — веб-интерфейс для работы с Prometheus.

В зависимости от утилиты управления брандмауэром, наши действия будут отличаться.

**Для iptables (как правило, на системах Deb)**
```bash
iptables -I INPUT -p tcp --dport 9090 -j ACCEPT
apt install iptables-persistent
netfilter-persistent save
```
**Для firewalld (как правило, на системах RPM)**
```bash
firewall-cmd --permanent --add-port={9090}/tcp
firewall-cmd --reload
```

## docker-compose
Созданим в корне папке **prometheus** (там у нас будут хранится различные данные для Prometheus)
Также создадим **docker-compose.yml**:
```yml
version: "3.9"
networks:
  app:
    driver: bridge

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus:/etc/prometheus/
    container_name: prometheus
    hostname: prometheus
    command:
      - --web.enable-lifecycle
      - --config.file=/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
    restart: unless-stopped
    environment:
      TZ: "Europe/Moscow"
    networks:
      - app
  node-exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    container_name: exporter
    hostname: exporter
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --collector.filesystem.ignored-mount-points
      - ^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)
    ports:
      - 9100:9100
    restart: unless-stopped
    environment:
      TZ: "Europe/Moscow"
    networks:
      - app
```
В данном примере мы создаем 2 сервиса — prometheus и node-exporter. Первый запускает сам Prometheus, а второй собирате локальную статистику и отправляет ее в Prometheus. Такой сервер называют **Exporter**.

Теперь в папке **prometheus** создадим файл с конфигурацией **prometheus.yml**

```yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    scrape_interval: 5s
    static_configs:
    - targets: ['node-exporter:9100']
```
В данном примере мы прописываем наш node-exporter в качестве цели (**target**).

Теперь можно запускать команду ```make init```

После запуска переходим по ссылке [http://127.0.0.1:9090](http://127.0.0.1:9090) и увидим web-интерфейс Prometheus

![web-интерфейс prometheus](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/prometheus/assets/web.png?raw=true)

Если перейти по ссылке [http://127.0.0.1:9100](http://127.0.0.1:9100), то мы увидим web-интерфейс NodeExporter

![web-интерфейс node exporter](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/prometheus/assets/node-exporter.png?raw=true)

## Базовая аутентификация
Как вы могли заметить, web-интерфейс Prometheus является открытым. Давайте добавим базовую аутентификацию (не лучший способ, но все же)

В папке **prometheus** создадим файл **web.yml**
```yml
basic_auth_users:
    admin: {bcrypt password}
```
Для генерации пароля можно воспользоваться [сервисом](https://bcrypt-generator.com/)
В качестве логина в примере используется **admin**
Теперь добавим флаг **--web.config.file** в наш **docker-compose.yml**
```yml
services:
  prometheus:
  # ...
  command:
  # ...
  - --web.config.file=/etc/prometheus/web.yml
```
Теперь после перезапуска при входе в web-интерфейс будет запрашиваться логин и пароль