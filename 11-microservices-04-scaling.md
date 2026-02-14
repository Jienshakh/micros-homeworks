
# Домашнее задание к занятию «Микросервисы: масштабирование»

Вы работаете в крупной компании, которая строит систему на основе микросервисной архитектуры.
Вам как DevOps-специалисту необходимо выдвинуть предложение по организации инфраструктуры для разработки и эксплуатации.

## Задача 1: Кластеризация

Предложите решение для обеспечения развёртывания, запуска и управления приложениями.
Решение может состоять из одного или нескольких программных продуктов и должно описывать способы и принципы их взаимодействия.

Решение должно соответствовать следующим требованиям:
- поддержка контейнеров;
- обеспечивать обнаружение сервисов и маршрутизацию запросов;
- обеспечивать возможность горизонтального масштабирования;
- обеспечивать возможность автоматического масштабирования;
- обеспечивать явное разделение ресурсов, доступных извне и внутри системы;
- обеспечивать возможность конфигурировать приложения с помощью переменных среды, в том числе с возможностью безопасного хранения чувствительных данных таких как пароли, ключи доступа, ключи шифрования и т. п.

Обоснуйте свой выбор.


## Решение
Использовать связку:

**Kubernetes + Ingress Controller + HPA + Secrets + ConfigMaps**

---

### Как это работает

- **Контейнеризация:**  
  Все сервисы упакованы в Docker-контейнеры и запускаются в Kubernetes Pod’ах

- **Обнаружение сервисов и маршрутизация:**  
  - Kubernetes Service — внутренняя балансировка  
  - Ingress Controller — внешний доступ и роутинг HTTP/HTTPS

- **Горизонтальное масштабирование:**  
  - ReplicaSet / Deployment увеличивают количество Pod’ов вручную  
  - HPA (Horizontal Pod Autoscaler) — автоматически по CPU/метрикам

- **Разделение внешних и внутренних ресурсов:**  
  - Ingress — точка входа извне  
  - ClusterIP Services — только внутри кластера

- **Конфигурация и секреты:**  
  - ConfigMap — переменные среды и настройки  
  - Secrets — пароли, токены, ключи (в зашифрованном виде)

---

###  Преимущества

- автоматическое масштабирование под нагрузкой  
- отказоустойчивость  
- стандарт де-факто для микросервисов  
- легко интегрируется с CI/CD  
- безопасное управление конфигурацией  

---

### Итог

Kubernetes предоставляет:

✔ контейнерную оркестрацию  
✔ сервис-дискавери и балансировку  
✔ авто- и горизонтальное масштабирование  
✔ сетевую изоляцию  
✔ безопасную работу с конфигурацией  

➡ оптимальное решение для enterprise-микросервисной архитектуры


## Задача 2: Распределённый кеш * (необязательная)

Разработчикам вашей компании понадобился распределённый кеш для организации хранения временной информации по сессиям пользователей.
Вам необходимо построить Redis Cluster, состоящий из трёх шард с тремя репликами.

### Схема:

![11-04-01](https://user-images.githubusercontent.com/1122523/114282923-9b16f900-9a4f-11eb-80aa-61ed09725760.png)

---

## Решение Redis Cluster + RedisInsight (Docker Compose)

Ссылка на файл [docker-compose](./11-microservices-04-scaling/docker-compose.yaml), запонен на основе этого [примера](https://github.com/bitnami/containers/blob/main/bitnami/redis-cluster/docker-compose.yml) и примеров с этой [страницы](https://hub.docker.com/r/bitnami/redis-cluster)
Образов на странице ```bitnami/redis-cluster``` нет, причину объясняют тут https://github.com/bitnami/charts/issues/36313#issuecomment-3355561730	

### Описание

Этот `docker-compose.yml` разворачивает:

1. **Redis Cluster**:
   - 12 нод (`redis-node-0` … `redis-node-11`)
   - 3 шарда с 3 репликами каждый
   - Настроен с паролем `bitnami`
   - Данные каждого узла хранятся в Docker volumes (`redis-cluster_data-*`)

2. **RedisInsight**:
   - Веб-интерфейс для работы с Redis
   - Данные хранятся в Docker volume `redisinsight-data`
   - Доступен на порту **5540**

---

### Структура Docker Compose

#### Volumes

| Volume | Назначение |
|--------|------------|
| `redis-cluster_data-0 … redis-cluster_data-11` | Данные Redis для каждой ноды |
| `redisinsight-data` | Данные RedisInsight (конфигурации, базы подключений) |


#### Services

| Service | Описание |
|---------|----------|
| `redis-node-0 … redis-node-11` | Ноды Redis Cluster |
| `redis-node-11` | Нода-инициатор кластера (`REDIS_CLUSTER_CREATOR=yes`) и порт 6379 наружу |
| `redis-insight` | Веб-интерфейс RedisInsight, порт 5540 |

---

### Переменные окружения

- `REDIS_PASSWORD=bitnami` — пароль для доступа к Redis
- `REDIS_NODES` — список всех нод кластера
- `REDIS_CLUSTER_REPLICAS=3` — количество реплик для каждого шарда
- `REDIS_CLUSTER_CREATOR=yes` — только для ноды-инициатора, создаёт кластер
- `REDISINSIGHT_ACCEPT_LICENSE=true` — согласие с лицензией RedisInsight

---

### Запуск

```bash
# 1. Запуск кластера
docker compose up -d

# 2. Проверка статуса
docker compose ps

# 3. Логи RedisInsight
docker ompose logs -f redis-insight
```

### Доступ

- RedisInsight: http://localhost:5540
- Redis Cluster: подключение через redis-cli или любой клиент с паролем bitnami на порт 6379 (через redis-node-11 или через внутренние ноды Docker)

### Работа с данными

- Для каждой ноды Redis данные хранятся в Docker volume redis-cluster_data-*.
- Для RedisInsight данные конфигурации хранятся в redisinsight-data.
- Если хотите видеть файлы на диске, замените Docker volumes на bind mounts:

```yaml
volumes:
  - ./redis-data-0:/bitnami/redis/data
```

###Полезные команды

```
# Подключение к Redis CLI на любой ноде
docker exec -it redis-node-11 redis-cli -a bitnami

# Просмотр всех ключей (в тестовом кластере)
keys *

# Остановка и удаление контейнеров
docker-compose down -v  # -v удаляет volumes
```

###Визуальная схема кластера

```
Shards: 3     Replicas: 3 per shard

Shard 0: redis-node-0 (master) 
         redis-node-1 (replica)
         redis-node-2 (replica)

Shard 1: redis-node-3 (master) 
         redis-node-4 (replica)
         redis-node-5 (replica)

Shard 2: redis-node-6 (master) 
         redis-node-7 (replica)
         redis-node-8 (replica)

Shard 3: redis-node-9 (master)
         redis-node-10 (replica)
         redis-node-11 (replica)

RedisInsight подключается к любому мастеру (например, redis-node-11)
```
### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
