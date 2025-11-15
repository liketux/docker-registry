# Private Docker Registry (registry:2) with Auth

Этот проект разворачивает приватный Docker Registry (**registry:2**) с включённой базовой авторизацией через `htpasswd`.
В продакшене registry обязательно должен работать через TLS. Обычно используют Nginx / Traefik как реверс-прокси c сертификатом (например, Let’s Encrypt). 
Подходит для локальной разработки, закрытых сетей, тестовых сред и оффлайн-инфраструктуры, тогда Registry работает напрямую по HTTP.

----------

## Возможности

-   Приватное хранение контейнерных образов

-   Авторизация через `htpasswd`

-   Простая структура каталогов

-   Возможность добавления/удаления пользователей

-   Очистка старых слоёв и мусора через `garbage-collect`


----------

## Структура проекта

```
.
├── docker-compose.yml
└── registry/
    ├── auth/ # htpasswd с пользователями
    ├── data/ # данные registry (образы) 
    └── config.yml # конфигурация registry
```

----------

## Запуск проекта

1.  Клонируйте репозиторий:


```
git clone https://github.com/username/private-registry.git cd private-registry
```

2.  Подготовьте директории:


```
mkdir -p registry/data registry/auth
```

3.  Создайте файл пользователей:


```
docker run --rm --entrypoint htpasswd httpd:2 -Bbn myuser mypassword > registry/auth/htpasswd
```

4.  Запустите Registry:


`docker-compose up -d`

Registry будет доступен на:

`http://localhost:5000`

----------

## Авторизация и работа с образом

### Логин:

```
docker login localhost:5000
```

### Push образа:

```
docker tag myapp:latest localhost:5000/myapp
docker push localhost:5000/myapp
```

### Pull образа:

```
docker pull localhost:5000/myapp
```

----------

## Добавление новых пользователей

Создание нового пользователя:

```
docker run --rm --entrypoint htpasswd httpd:2 -Bbn newuser newpass >> registry/auth/htpasswd
```

Перезапуск Registry не требуется — файл читается автоматически.

----------

## Удаление пользователей

Откройте `registry/auth/htpasswd` и удалите строку конкретного пользователя вручную.

Изменение применяется сразу.

----------

## Очистка старых образов (garbage collection)

Registry не удаляет слои автоматически. Чтобы удалить «висящие» слои после удаления тегов или push новых образов, выполняется GC.

1.  Остановите контейнер:


```
docker-compose down
```

2.  Запустите GC:


```
docker run --rm \
  -v $(pwd)/registry/data:/var/lib/registry \
  -v $(pwd)/registry/config.yml:/etc/docker/registry/config.yml \
  registry:2 garbage-collect /etc/docker/registry/config.yml
```

3.  Поднимите Registry снова:


```
docker-compose up -d
```

----------

## Удаление конкретного образа

1.  Найти digest:


```
curl -s http://localhost:5000/v2/myapp/tags/list
curl -v http://localhost:5000/v2/myapp/manifests/latest \
  -H "Accept: application/vnd.docker.distribution.manifest.v2+json"
```

2.  Удалить образ:


```
curl -X DELETE http://localhost:5000/v2/myapp/manifests/<digest>
```

3.  Выполнить garbage-collect (см. раздел выше).


----------

## Backup и восстановление

### Backup:

```
tar -czf registry-backup.tar.gz registry/data
```

### Restore:

```
tar -xzf registry-backup.tar.gz -C registry/data
```