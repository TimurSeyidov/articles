# Проект на React JS в Docker

**React** - это JavaScript-библиотека для создания пользовательских интерфейсов. Она была разработана компанией Facebook и используется для создания интерактивных одностраничных приложений. React позволяет создавать компоненты пользовательского интерфейса и управлять их состоянием, что делает его очень гибким и мощным инструментом для разработки веб-приложений.

**React Create App** - это инструмент командной строки, который упрощает процесс создания нового проекта на основе React. Он автоматически устанавливает все необходимые зависимости, создает структуру проекта и запускает сервер разработки. С его помощью можно быстро начать работу над новым проектом и сосредоточиться на разработке приложения.

## Автоматизация

Для более удобной работы, в корне проекта создадим **Makefile** для быстрого вызова команд.

**Makefile** - это файл, который содержит правила для автоматической сборки программ. Он используется в системе GNU Make и позволяет описывать зависимости между файлами проекта, а также команды для сборки этих файлов. С помощью Makefile можно автоматизировать процесс компиляции, линковки и других задач, связанных со сборкой программы.

Содержимое **Makefile**

```make
init: docker-down-clear frontend-clear docker-pull docker-build docker-up frontend-init
down: docker-down-clear frontend-clear
lint: frontend-lint

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

frontend-clear:
	docker run --rm -v ${PWD}/web:/app -w /app alpine sh -c 'rm -rf .ready build'

frontend-init: frontend-yarn-install frontend-ready

frontend-yarn-install:
	docker-compose run --rm frontend-node-cli yarn install

frontend-ready:
	docker run --rm -v ${PWD}/web:/app -w /app alpine touch .ready

frontend-lint:
	docker-compose run --rm frontend-node-cli yarn eslint
	docker-compose run --rm frontend-node-cli yarn stylelint

frontend-prettier:
	docker-compose run --rm frontend-node-cli yarn prettier

frontend-lint-fix:
	docker-compose run --rm frontend-node-cli yarn eslint-fix 

frontend-test:
	docker-compose run --rm frontend-node-cli yarn test --watchAll=false

frontend-test-watch:
	docker-compose run --rm frontend-node-cli yarn test
```

Подробнее о командах:
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

**frontend-clear**
Удаляет папку **build** из проекта, если она ранее была создана

**frontend-init**
Устанавливает пакеты, указаные в package.json

**frontend-yarn-install**
Устанавливает пакеты, указаные в package.json

**frontend-ready**
Создает файл **.ready**, который дает указание на запуск **yarn start**

**frontend-lint**
Запускает **eslint** и **stylelint**

**frontend-prettier**
Запускает **prettier**

**frontend-lint-fix**
Запускает **eslint-fix**

**frontend-test**
Запускает тесты и после завершения останавливается

**frontend-test-watch**
Запускает тесты в интерактивном режиме

## Подготовка конфигураций и Dockerfile

Так как приложение на React в процессе разработки запускает встроенный веб-сервер на 3000 порту, а также отслеживает изменения по websocket, подготовим конфигурации и docker-compose.yml для работы. В качестве пакетного менеджера используется **yarn**.

**Yarn** - это менеджер пакетов для **Node.js**, который используется для установки, обновления и удаления пакетов в проекте. Он был создан компанией Facebook и является альтернативой менеджеру пакетов npm. Yarn имеет ряд преимуществ перед npm, таких как более быстрая работа, улучшенная безопасность и поддержка асинхронной загрузки.

### Структура проекта

Проект будет иметь следующую структуру:
```
docker
- nginx
  - conf.d
    default.conf
  Dockerfile
- node
  Dockerfile
- web
docker-compose.yml
Makefile
```

В папке **docker** мы будем хранить Docker файлы сервисов и их настрйки

**docker/nginx/Dockerfile**

```bash
FROM nginx:1.19-alpine

COPY ./nginx/conf.d /etc/nginx/conf.d
WORKDIR /app
```
Собирается образ из nginx версии 1.19-alpine, копируется конфигурация и устанавливается в качестве рабочей директории папка **app**

**docker/nginx/conf.d/default.conf

```nginx
server {
    listen 8080;
    charset utf-8;
    server_tokens off;

    resolver 127.0.0.11 ipv6=off;

    add_header X-Frame-Options "SAMEORIGIN";

    location /health {
        add_header Content-Type text/plain;
        return 200 'alive';
    }

    location /ws {
        proxy_set_header  Host $host;
        proxy_set_header  Upgrade $http_upgrade;
        proxy_set_header  Connection "Upgrade";
        proxy_pass        http://frontend-node:3000;
        proxy_redirect    off;
    }

    location / {
        proxy_set_header  Host $host;
        proxy_pass        http://frontend-node:3000;
        proxy_redirect    off;
    }
}
```
Nginx будет работать на 8080 порте, проксировать http и websocket запросы на контейнер **frontend-node**

**docker/node/Dockerfile**
```bash
FROM node:18-alpine

WORKDIR /app
```
Собирается образ из node версии 18-alpine и устанавливается в качестве рабочей директории папка **app**

**docker-compose.yml**
```yml
version: "3.9"
networks:
  app:
    driver: bridge

services:
  frontend:
    container_name: frontend
    build:
      context: docker
      dockerfile: nginx/Dockerfile
    ports:
      - "9000:8080"
    depends_on:
      - frontend-node
    networks:
      - app
  frontend-node:
    container_name: frontend-node
    build:
      context: docker/node
    volumes:
      - ./web:/app
    command: sh -c "until [ -f .ready ] ; do sleep 1 ; done && yarn start"
    tty: true
    environment:
      - WDS_SOCKET_PORT=0
    networks:
      - app
  frontend-node-cli:
    container_name: frontend-cli
    build:
      context: docker/node
    volumes:
      - ./web:/app
    networks:
      - app
```
Снаружи приложение будет доступно по порту 9000. Сервис **frontend** будет основным и как раз будет заниматься проксированием запросов на **frontend-node**, где будет запущен **nodejs** сервер.
Сервис **frontend-node-cli** служит для ручного запуска node скриптов и используется в качестве инструмента.
Содержимое **Makefile** представлено в разделе **Автоматизация**

## Создание React проекта

Для создания приложения необходимо воспользоваться инструментом **create-react-app**

Давайте выполним команду для установки базового проекта
```bash
docker-compose run --rm frontend-node-cli npx create-react-app react-app
```
и на вопрос установки **create-react-app** ответим **y**
Теперь у нас в папке **web** должна появиться папка **react-app** с базовым проектом на React
Но нам нужно перенести эти файлы и папки на уровень вышу (в папке **web** непосредственно).
Перенесите содержимое папки **react-app** за исключением папки **node_modules** на уровень выше и удалите после папку **react-app**

Теперь отроем файл **web/.gitignore** и удалим лишние данные. По итогу содержимое должно быть следующим:
```
/node_modules
/coverage
/build
.idea
.DS_Store
yarn-debug.log*
yarn-error.log*
```
Теперь также в папке **web** создайте файл **.dockerignore** с этим же содержимым (чтобы эти файлы не попадали в сборку)

Теперь отроем файл **web/package.json** и сделаем следующие изменения:
1. Удалим ```"eject": "react-scripts eject"``` из секции **scripts**
2. После секции **dependencies** создадим секцию **devDependencies** и перенесем туда данные из секции **dependencies**:
```json
"devDependencies": {
    "@testing-library/jest-dom": "^5.17.0",
    "@testing-library/react": "^13.4.0",
    "@testing-library/user-event": "^13.5.0",
}
```
> версии пакетов могут отличаться

Если Вы работаете в IDE от компании Jetbrains, то там она может не понимать, почему Вы используете JSX формат в js файлах. Для решения проблемы можно поменять расширение файлов **App.js**, **App.test.js** и **index.js** в папке  **src** с **js** на **jsx**

Теперь можно запустить наш проект командой
```bash
make init
```
Переходим в браузере по ссылке [http://localhost:9000](http://localhost:9000) и увидим страницу с проектом React
![react web page](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/reactjs/assets/webpage.png?raw=true)

Если мы откроем панель разработки (F12 и Ctrl+Shift+I), перейти во вкладу **Network**, отфильтровать только отображение **ws**, а потом обновить страницу, мы обнаружим, что запрос на websocket успешно выполнен. А значит горячее обновление страницы работает:
![dev tools](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/reactjs/assets/devtools.png?raw=true)

Остановим наши контейнеры с помощью команды
```bash
make docker-down-clear
```

## Установка ESLint

**ESLint** - это инструмент для проверки и форматирования кода на JavaScript. Он помогает найти и исправить ошибки в коде, а также обеспечить его соответствие определенным стандартам кодирования.

Установим **ESLint**
```bash
docker-compose run --rm frontend-node-cli yarn add eslint --dev
```
После необходимо сконфигурировать ESLint:
```bash
docker-compose run --rm frontend-node-cli npx eslint --init
```
После консоль перейдет в интерактивный режим:
1. **How would you like to use ESLint**
   Выбираем **To check syntax, find problems, and enforce code style**
   Будем проверять не только синтаксические ошибки, но и проблемы с codestyle
2. **What type of modules does your project use?**
   Выбираем **JavaScript modules**
3. **Which framework does your project use?**
   Выбираем **React**
4. **Does your project use TypeScript?**
   Если вы хотите использовать TypeScript, выбираем **Yes**, иначе **No**
5. **Where does your code run?**
   Выберите необходимые варианты использования
6. **How would you like to define a style for your project?**
   Выберите **Use a popular style guide**
7. **Which style guide do you want to follow**
   Выберите тот styleguide, который хотите использовать
8. Если вы выбрали **Standart**, то на вопрос **What format do you want your config file to be in** можно ответить **JSON**
9. **Would you like to install them now?**
   Если мы выберем **Yes**, он установит зависимости через **npm**, а у нас **yarn**, поэтому выбираем **No**

Теперь скопируем список пакетов, которые он вывел в консоли и выполним команду
```bash
docker-compose run --rm frontend-node-cli yarn add eslint-plugin-react@latest eslint-config-standard@latest eslint@^8.0.1 eslint-plugin-import@^2.25.2 eslint-plugin-n@^16.0.0  eslint-plugin-promise@^6.0.0 --dev
``` 
>версии могут отличаться
Теперь добавим в файле **package.json** в секции **scripts** команды:
```json
"scripts": {
    ...
    "eslint": "eslint --ext .js,.jsx src",
    "eslint-fix": "eslint --fix  --ext .js,.jsx src"
}
```
В папке **web** у нас появился файл **.eslintrc.json**. Давайте его отредактируем
-  в секцию **env** добавим
   ```json
     "es6": true,
     "jest": true
   ```
   Это подскажет eslint, что не надо ругаться на файлы тестов Jest. Также подключаем поддержку стандарта EcmaScript 6
- добавим секцию **settings**
  ```json
    "settings": {
      "react": {
        "version": "18"
      }
    }
  ```
  > версию React можно посмотреть в **package.json**
- в секцию **rules** можно добавить правило игнорирования пробела перед описанием параметров функций, а также игнорирование проблем использования в jsx
  ```json
  "rules": {
    "space-before-function-paren": "off",
    "react/react-in-jsx-scope": "off",
    "react/jsx-uses-react": "off"
  }
  ```

## Установка Stylelint
**Stylelint** - это инструмент для проверки и соблюдения стилей CSS в коде. Он помогает находить и исправлять ошибки в коде, а также обеспечивает соблюдение единого стиля в команде.

Установка:
```bash
docker-compose run --rm frontend-node-cli yarn add stylelint stylelint-config-standard  --dev
```

Создаем в папке **web** файл **.stylelintrc.json** с содержимым:
```json
{
    "extends": [
        "stylelint-config-standart"
    ]
}
```
В **package.json** в секцию **scripts** добавляем
```json
"stylelint": "stylelint \"src/**/*.css\""
```

Теперь можем запустить команду ```make frontend-lint```, чтобы посмотреть работу наших линтеров.
![Работа линтера](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/reactjs/assets/lint_result.png?raw=true)

## Установка Prettier

**Prettier** - это утилита для форматирования кода, которая помогает поддерживать единый стиль кода и согласованность в команде. Она автоматически корректирует отступы, переносы строк, форматирование и другие аспекты кода для улучшения его читаемости и удобства сопровождения.

Установка:
```bash
docker-compose run --rm frontend-node-cli yarn add --dev --exact prettier
```

Также надо установить пакет для взаимодействия ESLint, Stylelint и Prettier
```bash
docker-compose run --rm frontend-node-cli yarn add --dev eslint-config-prettier eslint-plugin-prettier stylelint-config-prettier stylelint-prettier
```
В файле **.eslintrc.json** добавить:
```json
{
    ...
    "plugins": [
        ...,
        "plugin:prettier/recommended"
    ],
    "extends": [
        ...,
        "prettier"
    ],
    "rules": {
        ...,
        "prettier/prettier": "error"
    }
    ...
}
```
В файле **.stylelintrc.json** добавить:
```json
{
    ...,
    "plugins": [
        ...,
        "stylelint-prettier"
    ],
    "rules": {
        ...,
        "prettier/prettier": true
    },
    "extends": [
        ...,
        "stylelint-config-prettier"
    ]
}
```
В **package.json** в секцию **scripts** добавляем
```json
"prettier": "prettier --write \"**/*.+(js|jsx|json|css|md)\""
```

Теперь можно создать файл с именем **.prettierrc**:
```json
{
    "printWidth": 80,
    "arrowParens": "always",
    "semi": false,
    "tabWidth": 2,
    "singleQuote": true
}
```
Это всего лишь пример, подробнее [тут](https://prettier.io/docs/en/configuration)

Теперь после запуска ```make frontend-prettier```, у нас исправятся ошибки
![prettier](https://github.com/TimurSeyidov/articles/blob/main/pages/docker/reactjs/assets/prettier.png?raw=true)
