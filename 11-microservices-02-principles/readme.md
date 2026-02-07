# Как запускать
```
docker-compose up --build
```

# Как тестировать

## Login
Получить токен
```
curl -X POST -H 'Content-Type: application/json' -d '{"login":"bob", "password":"qwe123"}' http://localhost/v1/token
```
Пример
```
$ curl -X POST -H 'Content-Type: application/json' -d '{"login":"bob", "password":"qwe123"}' http://localhost/v1/token
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I
```

## Upload image
Использовать полученный токен для загрузки картинки
```
curl -X POST -H 'Authorization: Bearer <TODO: INSERT TOKEN>' -H 'Content-Type: octet/stream' --data-binary @<image_name.jpg> http://localhost/v1/upload
```
Пример
```
$ curl -X POST -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I' -H 'Content-Type: octet/stream' --data-binary @5ca2a2d650c753ada3f74ca7a4bd7aa0.jpg http://localhost/v1/upload
{"filename":"759399ed-3db9-4e20-87b4-03cb1755d38b.jpg"}
```

 ## Download image
Загрузить картинку и проверить что она открывается
```
curl -X GET -H 'Authorization: Bearer e<TODO: INSERT TOKEN>'  http://localhost/v1/user/<filename_from_previous_step> -o out.jpg
```
Пример
```
$ curl -X GET -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I'  http://localhost/v1/user/759399ed-3db9-4e20-87b4-03cb1755d38b.jpg -o out.jpg

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 13365  100 13365    0     0  4350k      0 --:--:-- --:--:-- --:--:-- 4350k

$ ls -la
-rw-r--r--. 1 dodinaev dodinaev 13365 Feb  1 06:52 5ca2a2d650c753ada3f74ca7a4bd7aa0.jpg
-rw-r--r--. 1 dodinaev dodinaev 13365 Feb  1 15:48 out.jpg
```

# Разбор кода nginx.conf

**Минимальная архитектура nginx-конфига**

Любой такой конфиг почти всегда состоит из:
1. upstream — описание микросервисов
2. server — входная точка
3. location — маршрутизация
4. механизм auth (header / subrequest / lua / basic)

**В nginx есть контексты:**

1. main (верхний уровень)
2. events
3. http
4. server (внутри http)
5. location (внутри server)

- ```server {}``` не может находиться на верхнем уровне, только внутри ```http {}```

Сколько процессов nginx будет обрабатывать запросы:
```
worker_processes 1;
```
Сколько одновременных соединений один процесс может держать:
```
events {
    worker_connections 1024; # сколько одновременных соединений один процесс может держать
}
```

Nginx слушает HTTP-запросы на порту 8080
```
http {
    
    server {
        listen 8080;
    ...
    }
...
}
```


```proxy_set_header Authorization $http_authorization:```

- Берёт заголовок Authorization от клиента
- Передаёт его во все проксируемые запросы, включая auth_request
- Без этого токен никогда не дойдёт до security, и всё будет 401

### POST /v1/register (регистрация):
```
location = /v1/register {
    proxy_pass http://security:3000/v1/user/;
}
```
**Логика**:
Запрос → сразу в security на создание пользователя (без проверки токена)

### POST /v1/token (получение токена):
```
location /v1/token {
    proxy_pass http://security:3000/v1/token;
}
```
**Логика**:
Запрос → сразу в security для генерации токена (логин)

### GET /v1/user (инфо о себе):
```
location /v1/user {
    auth_request /internal/token/validation;
    proxy_pass http://security:3000/v1/user;
}
```
**Логика**:

1. Запрос → /internal/token/validation (проверка токена)
2. Если 200 OK → запрос в security за данными пользователя
3. Если 401 → сразу ошибка клиенту

### POST /v1/upload (загрузка файла):
```
location /v1/upload {
    auth_request /internal/token/validation;
    proxy_pass http://uploader:3000/v1/upload;
}
```
**Логика**:

1. Проверка токена через internal location
2. Если ок → потоковая загрузка в uploader
3. ```client_max_body_size 50m``` - можно грузить до 50 МБ
4. ```proxy_request_buffering off``` - файл не буферизуется в nginx


### GET /v1/user/filename.txt (скачивание файла)
```
location ~ ^/v1/user/(.*)$ {
    auth_request /internal/token/validation;
    rewrite ^/v1/user/(.*)$ /data/$1 break;
    proxy_pass http://storage:9000;
}
```
**Логика**:

1. Проверка токена
2. rewrite преобразует /v1/user/myfile.txt → /data/myfile.txt
3. break - прекращает дальнейшие rewrite-правила
4. Запрос в storage (MinIO/S3) за файлом

### Внутренняя проверка токена (скрытая)
```
location /internal/token/validation {
    internal;
    proxy_pass http://security:3000/v1/token/validation;
}
```
**Логика**:

1. internal - доступно только из nginx (нельзя вызвать извне)
2. Передаёт заголовок Authorization в security
3. Security проверяет токен и возвращает 200/401


### Общий паттерн:

```
Публичный эндпоинт → auth_request → internal validation → 
→ (если 200) proxy_pass на целевой сервис
```
**Особенность**: Все запросы к user-данным (кроме register/token) требуют валидный токен в Authorization header
