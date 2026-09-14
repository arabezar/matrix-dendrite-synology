# Matrix Dendrite + LiveKit (HowTo)

Здесь описан процесс развертывания Matrix-сервера Dendrite с поддержкой видеозвонков через LiveKit на Synology NAS (DSM 7) с использованием Container Manager (Docker).

## Необходимые условия

- Synology DSM 7.x с админским доступом, в том числе и по SSH
- Установленный пакет `Container Manager`
- Наличие домена, возможность управлять им, а также валидного SSL-сертификата для используемых (под)доменов, иначе работать не будет.

## Ограничения

- Реализованная архитектура проверялась на белом статическом IP-адресе провайдера, как она будет вести себя на динамическом адресе - не знаю, проверяйте.
- Архитектура подразумевает использование малыми группами людей, например, дома, и не рассчитана на большие нагрузки, поэтому использована встроенная СУБД `SQLite`, которую при желании несложно заменить на `PostgreSQL` в контейнере.
- Клиент `Element X` пока [не поддерживается](https://github.com/element-hq/dendrite#dendrite) из-за отсутствия в `Dendrite` поддержки [MSC4186 (Simplified Sliding Sync)](https://github.com/matrix-org/matrix-spec-proposals/pull/4186) и [MSC3861 (Next-gen auth OIDC)](https://github.com/matrix-org/matrix-spec-proposals/pull/3861), подходят клиенты `Element Classic`, `SchildiChat` и т.д.

## Подготовка до развёртывания

### Проброс портов на роутере
Направьте на локальный IP Synology:
- TCP: 80 (HTTP), 443 (HTTPS) - традиционно
- UDP: 50100-50200 (LiveKit Media)

Не обязательно:
- TCP/UDP: 3478 (Matrix TURN)
- TCP: 7881 (LiveKit Signal/TCP fallback)
- UDP: 30000-40000 (LiveKit Media TURN relay)

### Настройка Synology Reverse Proxy
Для каждого поддомена создайте правила в `Панель управления` > `Портал для входа` > `Дополнительно` > `Обратный прокси`:
- `matrix.example.com:443` -> `http://localhost:18080` (WebSocket: ON)
- `auth.example.com:443` -> `http://localhost:18080` (WebSocket: ON)
- `livekit.example.com:443` -> `http://localhost:18080` (WebSocket: ON)
- `example.com:443` -> `http://localhost:18080` (WebSocket: ON) - не обязательно, но желательно, см. [Нюансы](#нюансы-развёртывания) ниже

## Развёртывание

Зайдите по SSH на Synology под админом, затем под `root`:
```bash
sudo -i
```

### Автоматическое развёртывание
Выполните команду загрузки и выполнения скрипта развёртывания (можно запускать в папке пользователя `root`):
```bash
curl -sLO https://raw.githubusercontent.com/arabezar/matrix-dendrite-synology/main/install.sh && chmod +x install.sh && ./install.sh
```
> [!NOTE]
> Что делает эта команда: загружает из репозитория скрипт и запускает его.
> 
> Что делает скрипт: проверяет возможность развёртывания, создаёт папку проекта в папке `docker`, запоминает ответы пользователя в файле `.env`, загружает шаблоны конфигурационных файлов в созданную папку, запускает команды контейнера по созданию ключа `Matrix`, ждёт создания проекта `Container Manager` пользователем, запускает команду создания администратора.
> 
> Скрипт работает исключительно в созданной подпапке `docker`, больше нигде и ничего не создаёт.
> 
> Скрипт можно остановить в любой момент нажатием `Ctrl+C`

Отвечайте на вопросы, следите за исполнением скрипта. В какой-то момент скрипт попросит перейти в `Container Manager`. Создайте и запустите проект в `Container Manager` > `Проект` > `Создать` > `Название проекта` (например, `matrix-dendrite`) + `Путь` > `Задать` (например, `/docker/matrix-dendrite`) > `Использовать существующий файл docker-compose.yml для создания проекта` > `Далее`…, после чего возвращайтесь к скрипту и задайте имя пользователя-администратора. Если не хотите отдельного админа, а сами хотите им быть, замените имя `admin` на своё, и создаваемый пользователь, т.е. Вы, сразу станет админом.
<details>
<summary>приблизительный лог работы скрипта</summary>

```bash
ash-4.4# curl -sLO https://raw.githubusercontent.com/arabezar/matrix-dendrite-synology/main/install.sh && chmod +x install.sh && ./install.sh
Установка Matrix Dendrite + LiveKit in Docker v0.0.1...
Проверка необходимых условий...
Задайте папку проекта [matrix-dendrite] (Enter - подтвердить):
Сбор параметров для развёртывания...
Основной домен [example.com] (Enter - подтвердить):
Домен Matrix [matrix.example.com] (Enter - подтвердить):
Домен LiveKit [matrixrtc-livekit.example.com] (Enter - подтвердить):
Домен LiveKit Auth [matrixrtc-auth.example.com] (Enter - подтвердить):
Секретная фраза (токен): Bla-bla-bla
Загрузка конфигурационных файлов...
Генерация ключа подписи Matrix...
Created private key file: /mnt/matrix_key.pem
Создайте в Container Manager проект matrix-dendrite, задайте путь /docker/matrix-denderite, запустите проект и продолжайте здесь... задайте имя пользователя-администратора [admin] (Enter - подтвердить):
Создание администратора Matrix...
Enter Password:
Confirm Password:
INFO[0023] Created account: admin (AccessToken: EiUdFUY-ft8DtqdzDnfTDjV2bvVs8jSUI-DATa8Okjc)
✅ Установка Matrix Dendrite завершена

```
</details>

### Создание пользователей
Последующих пользователей можно создавать командой (пароль будет запрошен дважды после запуска команды, запускать из папки проекта `docker`):

```bash
docker exec -it matrix-dendrite /usr/bin/create-account -config /etc/dendrite/dendrite.yaml -username <user_name>
```

### Нюансы развёртывания
> [!TIP]
> При необходимости для сервера можно задать имя как `matrix.example.com`, так и `example.com`, от этого будет зависеть суффикс полного имени пользователя: `@admin:matrix.example.com` или `@admin:example.com`. Если используется краткая форма `example.com`, то домен необходимо обязательно [пробросить через обратный прокси](#настройка-synology-reverse-proxy), чтобы сервер Matrix имел возможность взаимодействия с доменом, при этом все «лишние» запросы будут возвращены Synology для дальнейшей обработки, сервер Matrix обработает только предназначенные для него. Если вы столкнулись с «неправильным» поведением `example.com`, просто не делайте его проброс на прокси контейнеров, но ваши пользователи будут имет длинный суффикс `matrix.example.com`, что абсолютно не мешает никакой функциональности.

### Переустановка
> [!TIP]
> Если в момент установки что-то не устроило, всегда можно прервать процесс развёртывания и начать его снова (`./install.sh`). Любые конфигурационные файлы могут быть удалены, инсталлятор загрузит их и настроит повторно. С другой стороны, если какие-то файлы не удалять, то загружаться и настраиваться повторно они не будут, это же касается и ключа Matrix `matrix_key.pem`.

> [!IMPORTANT]
> Папка `db` хранит всю базу сообщений и пользователей вашего сервера, если удалить папку, вы потеряете и всех пользователей, и все сообщения, соответственно, придётся заново регистрировать всех пользователей, включая администратора. Ключ Matrix `matrix_key.pem` также не имеет смысла удалять без удаления папку `db`, т.к. при удалении ключа вы потеряете доступ к своему серверу, а новый сгенерированный ключ к старой базе не подойдёт.

> [!CAUTION]
> Файл `.env` хранит все настройки, введённые пользователем, и автоматически подхватывается инсталлятором. Также этот файл играет важную роль для автонастройки конфигурационных файлов `compose.yaml` и `proxy.conf.template`, однако, если в файле вручную изменить какие-то переменные, они не будут автоматически обновлены в конфигурационных файлах `dendrite.yaml` и `livekit.yaml`. Самым простым и правильным способом изменения переменных в `.env` файле является удаление файлов `dendrite.yaml` и `livekit.yaml` и повторный запуск инсталлятора `./install.sh`, где вы можете переопределить переменные и пересоздать проект `Container Manager`. На последнем шаге (на паузе до создания администратора) вы можете прервать исполнение инсталлятора (Ctrl+C), а база останется нетронутой.

### Удаление
Если по каким-то причинам требуется удалить развёрнутый проект, следует для проекта в `Container Manager` последовательно вызвать два пункта меню `Действия`: `Очистить` (будут остановлены, если запущены, и удалены контейнеры) и `Удалить` (будет удалён проект). Все данные и настройки в папке `matrix-dendrite` останутся и удалены не будут, что позволит в любой момент пересоздать проект, указав в качестве проекта папку `matrix-dendrite`.

## Тестирование сервера
Есть множество вариантов, например, [testmatrix](https://codeberg.org/spaetz/testmatrix), - после установки инструмента в консоли `VS Code` введите команду:
```bash
testmatrix example.com
```
<details>
<summary>инструмент укажет на проблемы с сервером</summary>

```bash
Testing server example.com
  Federation url: https://matrix.example.com:443
✔ Server well-known exists
✔ Client well-known has proper CORS header
  Client url: https://example.com
  Adding livekit service URL: https://auth.example.com
✔ Server version: Dendrite (0.15.2+e546df2)
✔ Federation API endpoints seem to work fine
✔ Client API endpoints seem to work fine
  QR code login is disabled (MSC 4108)
  Public room directory is enabled
✔ MatrixRTC SFU configured
  JWTauth healtz url: https://auth.example.com
  jwt has no CORS header (that is OK)
✔ JWTauth responds
✔ jwt /sfu/get without auth returns (405). This is good!
  jwt: no credentials passed, not trying authed requests
𐄂 MatrixRTC configured but delayed events turned off (MSC4140). BAD!
  No room summaries (MSC3266) (unstable) support
𐄂 Direct open registration might not be forbidden!
```
> Не обращайте внимания на крестики внизу: при включении MSC4140 в `Dendrite`, последний начинает безбожно глючить, - видео рвётся сразу после подключения; а второй крестик о регистрации не соответствует действительности, т.к. в конфигурации открытая регистрация запрещена.
</details>
Можно протестировать и авторизацию с указанием пользователя и токена (который можно найти в UI клиента после регистрации пользователя; например, в Element Web - клик на пользователе > Все настройки > Помощь и о программе > Токен доступа)

```bash
testmatrix -u admin -t <token> example.com
```

### Тестирование в браузере
Теперь можно зайти на свой сервер через коиента [Element Web](https://app.element.io): `Войти` > `Изменить` сервер на свой `example.com` > `Продолжить` > Ввести имя созданного ранее пользователя и пароль > `Войти`.

## Технологии

- [Synology](https://www.synology.com/)    
- [Docker](https://www.docker.com/)
- [Matrix](https://matrix.org), [Matrix Dendrite](https://github.com/matrix-org/dendrite), [Continuwuity](https://continuwuity.org), [Matrix server setup using Ansible and Docker](https://github.com/spantaleev/matrix-docker-ansible-deploy), [Adding bridges](https://gitlab.com/rogs/dendrite-docker-bridges)
- [Клиенты Element](https://element.io/download)

## Вклад

- Предложения и замечания категорически приветствуются [здесь](https://github.com/arabezar/matrix-dendrite-synology/discussions)
