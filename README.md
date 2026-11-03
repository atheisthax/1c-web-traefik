Ниже готовый документ, его можно целиком скопировать в файл, например `README-1c-traefik.md`.

Пара допущений, проверьте их по своему конфигу:
- бэкенд оставлен на **HTTPS без проверки сертификата** (`insecureSkipVerify`), вариант с HTTP описан как альтернатива;
- итоговая схема **скрывает `/1c_prod/ru_RU`** через `addPrefix`, раз запросы у вас идут на `/e1cib/users`;
- список пользователей **подменяется плагином `staticresponse`**, блокировка через 403 описана как альтернатива.

---

# Публикация веб-клиента 1С через Traefik (reverse proxy + Let's Encrypt)

## 1. Задача

Опубликовать в интернет веб-клиент 1С, развёрнутый на Windows-сервере с Apache, так, чтобы:

- снаружи был открыт **только порт 443**;
- использовался бесплатный сертификат **Let's Encrypt** с автопродлением;
- пользователь открывал `https://1c.example.com/` без пути публикации `/1c_prod` в адресе;
- в форме входа веб-клиента, открытого через прокси, **не показывался полный список пользователей**, а у клиентов, работающих напрямую с Apache, всё осталось как было.

## 2. Схема

```
Интернет
   │  HTTPS :443 (сертификат Let's Encrypt)
   ▼
┌──────────────────────────┐
│ Traefik (Docker)         │
│  - TLS-терминация        │
│  - ACME TLS-challenge    │
│  - addPrefix /1c_prod/.. │
│  - подмена e1cib/users   │
└──────────────────────────┘
   │  HTTPS (самоподписанный серт, без проверки)
   ▼
┌──────────────────────────┐
│ Windows: Apache 2.4      │
│ + модуль 1С (wsap24)     │
│ публикация /1c_prod      │
│ 192.168.1.10:443         │
└──────────────────────────┘
```

## 3. Исходные данные

| Параметр | Значение (замените на свои) |
|---|---|
| Внешнее имя | `1c.example.com` |
| Адрес Apache с 1С | `192.168.1.10:443` (HTTPS, самоподписанный сертификат) |
| Путь публикации | `/1c_prod` |
| Версия платформы | 8.5.1.1529 |
| Traefik | `v3.1` (Docker) |
| Email для Let's Encrypt | `admin@example.com` |

> ⚠️ Путь публикации должен совпадать **символ в символ**. В ходе настройки встречалась кириллическая «с» в `1с_prod`. Путь в конфиги лучше копировать из реального запроса в DevTools, а не набирать вручную.

## 4. Требования

- DNS: A-запись `1c.example.com` указывает на внешний IP хоста с Traefik **до первого запуска**.
- Порт **443** из интернета проброшен напрямую на Traefik.
- Перед Traefik нет ничего, что терминирует TLS (например, проксирование Cloudflare). Иначе TLS-challenge не пройдёт.
- Если Traefik и Apache на одном хосте, Apache не должен занимать 443 (перевести на другой порт).
- У Traefik есть доступ в интернет: ему нужен Let's Encrypt и загрузка плагина при старте.

## 5. Структура файлов

```
traefik/
├── docker-compose.yml
├── traefik.yml          # статическая конфигурация
├── dynamic.yml          # роутеры, middleware, сервисы
└── letsencrypt/         # сюда Traefik сам положит acme.json
```

## 6. docker-compose.yml

```yaml
services:
  traefik:
    image: traefik:v3.1
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./dynamic.yml:/etc/traefik/dynamic.yml:ro
      - ./letsencrypt:/letsencrypt
    # Если Apache на этом же хосте (вне Docker), раскомментировать
    # и в dynamic.yml указать https://host.docker.internal:<порт>
    # extra_hosts:
    #   - "host.docker.internal:host-gateway"
```

## 7. traefik.yml (статическая конфигурация)

Изменения в этом файле применяются **только после перезапуска** контейнера.

```yaml
entryPoints:
  websecure:
    address: ":443"
    http:
      # ОБЯЗАТЕЛЬНО для 1С, см. раздел 9.3
      encodeQuerySemicolons: true
    transport:
      respondingTimeouts:
        readTimeout: 600s
        writeTimeout: 600s
        idleTimeout: 600s

certificatesResolvers:
  le:
    acme:
      email: admin@example.com
      storage: /letsencrypt/acme.json
      tlsChallenge: {}

providers:
  file:
    filename: /etc/traefik/dynamic.yml
    watch: true

experimental:
  plugins:
    staticresponse:
      moduleName: github.com/jdel/staticresponse
      version: v0.0.1   # зафиксированная версия, обновлять осознанно

log:
  level: INFO

# Access-лог: полезен для диагностики, можно выключить
accessLog: {}
```

## 8. dynamic.yml (динамическая конфигурация)

Подхватывается на лету (`watch: true`).

```yaml
http:
  routers:
    # 1. Подмена списка пользователей формы входа (наивысший приоритет)
    onec-users:
      rule: "Host(`1c.example.com`) && PathRegexp(`^(/1c_prod/[a-z]{2}_[A-Z]{2})?/e1cib/users$`)"
      entryPoints: [websecure]
      middlewares: [users-list]
      service: noop@internal
      priority: 100
      tls:
        certResolver: le

    # 2. Запросы, которые уже содержат /1c_prod, уходят на бэкенд как есть
    onec-direct:
      rule: "Host(`1c.example.com`) && PathPrefix(`/1c_prod`)"
      entryPoints: [websecure]
      service: onec
      priority: 20
      tls:
        certResolver: le

    # 3. Всё остальное: добавляем скрытый префикс публикации
    onec:
      rule: "Host(`1c.example.com`)"
      entryPoints: [websecure]
      service: onec
      middlewares: [onec-prefix]
      priority: 10
      tls:
        certResolver: le

  middlewares:
    onec-prefix:
      addPrefix:
        prefix: "/1c_prod/ru_RU"

    users-list:
      plugin:
        staticresponse:
          StatusCode: 200
          Body: "admin\r\nuser1\r\n"   # нужные пользователи через \r\n, или "" для пустого списка
          Headers:
            Content-Type: "text/plain; charset=utf-8"

  services:
    onec:
      loadBalancer:
        passHostHeader: true
        serversTransport: apache
        servers:
          - url: "https://192.168.1.10:443"

  serversTransports:
    apache:
      insecureSkipVerify: true
```

## 9. Пояснения к ключевым решениям

### 9.1. Только 443 и TLS-challenge

Let's Encrypt проверяет владение доменом одним из способов: HTTP-01 (порт 80), DNS-01 или TLS-ALPN-01 (порт 443). Выбран `tlsChallenge`, чтобы не открывать 80-й порт.

Следствия:
- редиректа HTTP → HTTPS нет, пользователи сразу открывают `https://...`;
- `acme.json` Traefik создаёт сам с правами 600;
- продление выполняется автоматически примерно за 30 дней до истечения срока.

> Если когда-нибудь будете ограничивать `alpnProtocols` в TLS-опциях, обязательно оставьте `acme-tls/1`, иначе продление сломается.

### 9.2. Бэкенд: HTTPS без проверки сертификата

У Apache самоподписанный сертификат. Чтобы Traefik его проверял, нужен файл сертификата в `rootCAs` и корректный `serverName`, совпадающий с **SAN**. Go с версии 1.15 игнорирует CN, поэтому старые сертификаты без SAN не проходят.

Посмотреть SAN:

```bash
openssl s_client -connect 192.168.1.10:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

Так как трафик между Traefik и Apache идёт по доверенной сети, выбрано `insecureSkipVerify: true`.

**Альтернатива: бэкенд по HTTP.** Отдельный HTTP-виртуалхост на Apache (например, `:8080`) и `url: "http://192.168.1.10:8080"` без `serversTransport`. Проверьте, что в конфиге Apache нет глобального редиректа на HTTPS, иначе запросы зациклятся.

### 9.3. Точка с запятой в URL (главная проблема)

**Симптом:** напрямую через Apache всё работает, а через Traefik сразу после входа веб-клиент падает. В DevTools запрос

```
/e1cib/modules/defs?...&id=urn:module:md:...@property='...';version='...'
```

возвращает `400` с ошибкой компиляции:

```
Ошибка инициализации модуля: Тип не определен (ОписаниеОповещения)
```

**Причина:** Traefik написан на Go, а Go начиная с 1.17 считает `;` в query-строке недопустимым и **молча отбрасывает** такие параметры. Часть `;version=...` не доходит до 1С, и модуль компилируется в неверном контексте. Это видно и по ответу: в `moduleId` нет `version`.

**Решение:** в entrypoint указать

```yaml
http:
  encodeQuerySemicolons: true
```

Тогда `;` передаётся как `%3B`, и 1С получает параметр целиком. **Эту опцию не убирать.**

### 9.4. Скрытие пути публикации

Middleware `addPrefix` с префиксом `/1c_prod/ru_RU` дописывает путь к запросу перед отправкой на Apache:

```
браузер:  https://1c.example.com/e1cib/...
Apache:   https://192.168.1.10/1c_prod/ru_RU/e1cib/...
```

- Языковой сегмент `ru_RU` включён в префикс сразу, иначе 1С сама редиректит на `/1c_prod/ru_RU/`, и путь становится виден.
- Роутер `onec-direct` (приоритет 20) нужен для абсолютных ссылок, которые 1С может сгенерировать сама (`/1c_prod/...`). Без него получилось бы `/1c_prod/1c_prod/...`.
- Минус: язык фиксирован. Английский интерфейс доступен по `https://1c.example.com/1c_prod/en_US/`.
- Тонкий клиент можно подключать как `https://1c.example.com/`, так и `https://1c.example.com/1c_prod`.

> Правило роутера сравнивается с **исходным** путём от браузера, до применения middleware. Поэтому в правилах фигурирует `/e1cib/...`, а не `/1c_prod/ru_RU/e1cib/...`.

**Запасной вариант**, если скрытие начнёт что-то ломать: временный редирект только корня.

```yaml
  middlewares:
    onec-root-redirect:
      redirectRegex:
        regex: "^https://1c\\.example\\.com/?$"
        replacement: "https://1c.example.com/1c_prod/"
        permanent: false
```

### 9.5. Скрытие списка пользователей только для веб-доступа

Флаг «Показывать в списке выбора» у пользователя ИБ действует на все клиенты сразу. Отдельной настройки для веб-клиента в платформе нет.

Веб-клиент получает список отдельным запросом `.../e1cib/users`, а ответ на него — простой текст, по одному имени в строке:

```
admin
user1
user2
```

Роутер `onec-users` (приоритет 100) перехватывает этот запрос, и вместо Apache ответ формирует плагин `staticresponse`. Клиенты, работающие напрямую с Apache, видят полный список.

Получить оригинальный ответ как образец (на сервере с Apache):

```powershell
curl.exe -k -i https://localhost/1c_prod/ru_RU/e1cib/users
```

> Не используйте `curl -I`: это метод HEAD, на него 1С отвечает `405`.

Старая инструкция с infostart (через `.htaccess` + `RewriteRule` на `users.txt`) не заработала: в Apache обычно `AllowOverride None`, публикацию обрабатывает модуль 1С, а редирект вёл в корень сайта.

**Альтернатива без плагина:** заблокировать запрос (форма покажет пустое поле логина).

```yaml
  middlewares:
    deny-all:
      ipAllowList:
        sourceRange:
          - "127.0.0.1/32"
```

## 10. Запуск и проверка

```bash
cd traefik
docker compose up -d
docker compose logs -f traefik
```

Проверки:

```bash
# сертификат выдан Let's Encrypt
curl -vI https://1c.example.com/ 2>&1 | grep -i issuer

# подмена списка пользователей (GET и POST)
curl -i https://1c.example.com/e1cib/users
curl -i -X POST -d "" https://1c.example.com/e1cib/users

# версия Traefik
docker compose exec traefik traefik version
```

В браузере (лучше в режиме инкогнито):
1. `https://1c.example.com/` открывает форму входа, в адресе нет `/1c_prod`.
2. В списке только нужные пользователи, вход по логину, введённому вручную, работает.
3. После входа нет ошибок, открываются формы и отчёты, загружаются файлы.
4. Тонкий клиент подключается по `https://1c.example.com/` или `https://1c.example.com/1c_prod`.

## 11. Диагностика

| Симптом | Причина / что проверить |
|---|---|
| Сертификат не выпускается | 443 недоступен снаружи, неверная A-запись, перед Traefik стоит TLS-прокси |
| `502 Bad Gateway` | Traefik не достучался до Apache: адрес, порт, фаервол |
| `Тип не определен (ОписаниеОповещения)`, `400` на `modules/defs` | Нет `encodeQuerySemicolons: true` (раздел 9.3) |
| Mixed content | Бэкенд по HTTP, 1С строит `http://`-ссылки. Вернуть HTTPS на бэкенде |
| Двойной префикс `/1c_prod/1c_prod` | Нет роутера `onec-direct` |
| Список пользователей не подменяется | Не совпадает путь в `PathRegexp` или не задан `priority: 100` |
| Плагин не работает | Нет доступа в интернет при старте или неверная версия плагина |

Инструменты:
- **DevTools → Network** (с галкой Preserve log): URL, статус и тело упавшего запроса;
- **access-лог Traefik**: для подробностей включить `format: json` и сравнить `OriginStatus` (ответ Apache) с `DownstreamStatus` (ответ клиенту);
- **access-лог Apache**: проверить, в каком виде запрос дошёл до бэкенда.

## 12. Обслуживание

- **Изменения `traefik.yml`** применяются только после `docker compose restart traefik`. `dynamic.yml` подхватывается сам.
- **Список пользователей в заглушке** с 1С не синхронизируется. При изменении состава пользователей правьте `Body` вручную.
- **После обновления платформы 1С** проверьте вход, работу веб-клиента и путь запроса `e1cib/users`: внутренний протокол может поменяться.
- **Обновление Traefik и плагина**: версии зафиксированы, обновлять осознанно и проверять после обновления.
- **Сертификат** продлевается автоматически. Периодически проверяйте `docker compose logs traefik` на ошибки ACME.
- **Резервная копия**: каталог `traefik/` целиком, включая `letsencrypt/acme.json`.

## 13. Безопасность

- Скрытие списка пользователей — косметика, а не защита. У всех пользователей ИБ должны быть надёжные пароли, пользователей без пароля с правом входа быть не должно.
- `insecureSkipVerify` допустим только при доверенной сети между Traefik и Apache.
- По возможности стоит рассмотреть дополнительную защиту: ограничение по IP (`ipAllowList`), OpenID Connect или VPN.

---

Если нужно, могу сделать вариант документа под бэкенд по HTTP или с блокировкой списка через 403 вместо плагина.