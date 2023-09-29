# Grafana

**Grafana** - это инструмент для визуализации и анализа данных, разработанный компанией Grafana Labs. Он позволяет создавать различные типы графиков и диаграмм для отображения метрик, собираемых системами мониторинга, такими как Prometheus. Grafana может работать с различными источниками данных, включая базы данных, очереди сообщений, API сервисов и другие. Она также имеет встроенный язык запросов, который позволяет создавать сложные визуализации и анализировать большие объемы данных.

**Grafana** имеет удобный интерфейс для создания и редактирования графиков. Она поддерживает множество типов графиков, таких как линейные графики, столбчатые графики, круговые диаграммы, тепловые карты и другие. **Grafana** также позволяет настраивать цвета, шрифты, размеры графиков и другие параметры для создания красивых и информативных визуализаций. Она также поддерживает экспорт графиков в различные форматы, такие как PNG, JPEG, SVG и другие, что позволяет использовать Grafana не только для анализа данных, но и для создания отчетов и презентаций.

Для начала создадим в корне проекта **Makefile** ([подробнее тут](../makefile/README.md))

Теперь выполним предварительные действия.

## Настройка брандмауэра
Если на вашем сервере работает брандмауэр, необходимо открыть порт:
- **3000/tcp** — веб-интерфейс для работы с Grafana.

В зависимости от утилиты управления брандмауэром, наши действия будут отличаться.

**Для iptables (как правило, на системах Deb)**
```bash
iptables -I INPUT -p tcp --dport 3000 -j ACCEPT
apt install iptables-persistent
netfilter-persistent save
```
**Для firewalld (как правило, на системах RPM)**
```bash
firewall-cmd --permanent --add-port={3000}/tcp
firewall-cmd --reload
```

## docker-compose
Созданим в корне папке **grafana** (там у нас будут хранится различные данные для Grafana)
Также создадим **docker-compose.yml**:
```yml
version: "3.9"
networks:
  app:
    driver: bridge

services:
  grafana:
    image: grafana/grafana
    user: root
    depends_on:
      - prometheus
    ports:
      - 3000:3000
    volumes:
      - ./grafana:/var/lib/grafana
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    container_name: grafana
    hostname: grafana
    restart: unless-stopped
    environment:
      TZ: "Europe/Moscow"
    networks:
      - app
```

Теперь можно запускать команду ```make init```

После запуска переходим по ссылке [http://127.0.0.1:3000](http://127.0.0.1:3000) и увидим web-интерфейс Grafana.
Логин и пароль: **admin**/**admin**
После **Grafana** предложит сменить пароль

![web-интерфейс grafana](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/grafana/assets/web.png?raw=true)


## Интеграция с Prometheus
Перед тем, как делать интеграцию, выполните действия из [этой статьи](../prometheus/README.md)

Немного модифицируем настройки **prometheus.yml**
Добавим еще один **target**
```yml
  - job_name: 'prometheus'
    scrape_interval: 15s
    static_configs:
      - targets: ['localhost:9090']
    basic_auth:
      username: <user>
      password: <password>
```
Если Вы не настраивали базовую аутентификацию, то секцию **basic_auth** можно удалить.

После переходим по ссылке [http://127.0.0.1:3000](http://127.0.0.1:3000)

Кликаем по иконке **Connections** - **Data Sources**:

![Data Source](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/grafana/assets/data_source.png?raw=true)

и нажимаем **Add data source**

Среди списка источников данных находим и выбираем **Prometheus**, кликнув по **Select** переходим к настройкам:

**HTTP**
- **Prometheus server URL**
  http://prometheus:9090
- **Auth**
  Если вы настраивали базовую аутентификацию в Prometheus, включите переключатель **Basic Auth** и введите логин/пароль

После нажмите **Save and Test**

![save and test](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/grafana/assets/save_test.png?raw=true)

Добавим дашборд для мониторинга с node exporter. Для этого уже есть готовый вариант.
Выбираем в левой панели **Dashboards** и выбираем **Import**

![Import](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/grafana/assets/prometheus_import.png?raw=true)

Вводим идентификатор дашборда. Для Node Exporter это **1860**.
Нажимаем **Load** — **Grafana** подгрузит дашборд из своего репозитория — выбираем в разделе **Prometheus** наш источник данных и кликаем по **Import**.

![prometheus source](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/grafana/assets/import_dashboard.png?raw=true)

В результате мы увидим Dashboard с метриками, которые нам отправил **Node exporter**

![Chart](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/grafana/assets/exporter_chart.png?raw=true)