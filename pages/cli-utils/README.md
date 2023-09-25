# Подборка консольных утилит
[назад к оглавлению](/README.md)

Данный список содержит список консольных утилит, которые могут упростить работу.
> Работают в операционных системах семейства Linux и Unix
> Для установки в MacOS рекомендуется использовать [Homebrew](https://brew.sh/)
> Примеры приведены с использованием пакетного менеджреа **apt**

## Midnight Commander
Сайт: [midnight-commander.org](https://midnight-commander.org/)

Я думаю, что в наше время не один разработчик, использующий Linux/Unix не обходится к без этого инструмента. Это - консольный файловый менеджер, который вдохновлен продуктом от компании Norton

Установка
```bash
apt install mc
```

![mc](https://github.com/TimurSeyidov/articles/blob/main/pages/cli-utils/assets/mc.png?raw=true)


## htop
Сайт: [htop.dev](https://htop.dev)

Данная утилита является стандартом "де-факто" уже на всех серверах (да и ПК) все линуксоидов. Это - продвинутый менеджер процессов, который уделывает классический **top** по всем фронтам

Установка
```bash
apt install htop
```

![htop](https://github.com/TimurSeyidov/articles/blob/main/pages/cli-utils/assets/htop.png?raw=true)


## OhMyZsh 
Сайт: [ohmyz.sh](https://ohmyz.sh)
> работает поверх Zsh

Если Вы давно поняли, что классического **bash** Вам мало (и, возможно, вы уже перешли на **zsh**), то Вы можете пойти дальше и воспользоваться **OhMyZsh**

Как заявляют разработчики, это - framework для управления Вашими zsh конфигурациями. Позволяет настроить горячие клавиши, внешний вид и много другое в Вашей консоли.

Установка Zsh
```bash
apt install zsh
```

Установка OhMyZsh
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

![ohmyzsh](https://github.com/TimurSeyidov/articles/blob/main/pages/cli-utils/assets/ohmyzsh.png?raw=true)

## Bat
Репозиторий: [github.com/sharkdp/bat](https://github.com/sharkdp/bat)

Все мы прекрасно знаем и пользуемся командой **cat** для вывода содержимого файлов.
Но разработчики пошли далше и написали утилиту **bat** (не путать с почтовым клиентом), которая тоже выводит содержимое файлов, но также может давать дополнительные функции:
- подсветка синтаксиса для файлов с кодом
- постраничный просмотр
- темы оформления

Установка
```bash
apt install bat
```

![bat](https://github.com/TimurSeyidov/articles/blob/main/pages/cli-utils/assets/bat.png?raw=true)

## Exa
Репозиторий: [github.com/ogham/exa](https://github.com/ogham/exa)

Утилита **Exa** дополняет утилиту **ls**, предоставляя возможность подсвечивать типы файлов при выводе, а также выводить файлы в виде дерева

Установка
```bash
apt install exa
```

![exa](https://github.com/TimurSeyidov/articles/blob/main/pages/cli-utils/assets/exa.png?raw=true)

## fd
Репозиторий: [github.com/sharkdp/fd](https://github.com/sharkdp/fd)

Эта утилита - простая альтернатива встроенной утилиты **find**. Конечно, она не содержит всех возможностей **find**, но при этом покрывает большую часть стандартных кейсов, с более удобным синтаксисом

Установка
```bash
apt install fd-find
```

![fd](https://github.com/TimurSeyidov/articles/blob/main/pages/cli-utils/assets/fd.png?raw=true)

## httpie
Сайт: [httpie.io](https://httpie.io/)

Если Вы тот, кто берет от cURL только выполнение запросов или же Вам необходимо потестировать Ваш api, то **httpie** отлично пополнит Вашу коллекцию. Он также выполняет запросы, имеет настройки и выводит результат в отформатированном в зависимости от типа файлов виде.

[Установка](https://httpie.io/docs/cli/installation) описана тут

![httpie](https://github.com/TimurSeyidov/articles/blob/main/pages/cli-utils/assets/httpie.png?raw=true)
