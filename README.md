# Домашнее задание: Микросервисы и масштабирование

## Задача 1: Кластеризация

### Предлагаемое решение: Docker Swarm + Portainer + Docker Secrets

  Для организации инфраструктуры разработки и эксплуатации микросервисов предлагается использовать **Docker Swarm** как основной инструмент оркестрации контейнеров.

### Почему выбран Docker Swarm?

Docker Swarm — это встроенное решение для кластеризации в экосистеме Docker. Оно обладает следующими преимуществами для новичков:

  - **Простота настройки** — инициализация кластера выполняется одной командой (`docker swarm init`)
  - **Единый синтаксис** — используется тот же `docker-compose.yml`, что и для локальной разработки
  - **Встроенные механизмы** — не требует установки дополнительных компонентов
  - **Понятная документация** — большое количество примеров и готовых решений

### Соответствие требованиям

#### 1. Поддержка контейнеров
  Docker Swarm полностью поддерживает работу с контейнерами Docker. Любое приложение, упакованное в Docker-образ, может быть развернуто в кластере.

#### 2. Обнаружение сервисов и маршрутизация запросов
  - **Встроенный DNS-сервер** — каждому сервису автоматически назначается DNS-имя вида `service_name.stack_name`
  - **Встроенный балансировщик нагрузки** — распределяет входящие запросы между всеми репликами сервиса
  - **Ingress-сеть** — позволяет публиковать сервисы наружу с автоматической маршрутизацией

#### 3. Горизонтальное масштабирование
  Масштабирование выполняется одной командой:

    docker service scale service_name=5

  Это увеличивает количество реплик сервиса с текущего значения до указанного.

#### 4. Автоматическое масштабирование
  Docker Swarm не имеет встроенного автопоиска, но может быть дополнен:
  
  Интеграция с Docker API — сторонние инструменты (например, Prometheus + Alertmanager) могут отправлять команды на масштабирование
  
  Простой скриптовый подход — bash-скрипт, который проверяет нагрузку и изменяет количество реплик

#### 5. Разделение ресурсов (внешние/внутренние)
  Разделение реализуется через пользовательские сети (networks):
    
    Сеть frontend — доступна извне, содержит публичные сервисы (веб-интерфейс, API Gateway)
    
    Сеть backend — изолирована, содержит внутренние сервисы (базы данных, кеши, очереди)
    
    Сеть database — дополнительная изоляция для компонентов хранения данных
  
  Пример конфигурации:

      networks:
    frontend:
      external: true
    backend:
      internal: true
    database:
      internal: true

#### 6. Конфигурирование через переменные среды и безопасное хранение

  Переменные среды — передаются через параметр environment в docker-compose.yml
  
  Файлы .env — используются для хранения переменных окружения вне кода приложения
  
  Docker Secrets — безопасное хранение чувствительных данных (пароли, ключи API, сертификаты):

    # Создание секрета
    echo "supersecretpassword" | docker secret create db_password -
    
    # Использование в сервисе
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password

## Задача 2: Распределённый кеш (Redis Cluster)

  Решение: Redis Cluster на базе Docker
  Для организации распределённого кеша предлагается использовать Redis Cluster в режиме "3 шарды × 3 реплики".
  
  Архитектура кластера
  Каждая шарда состоит из:
  
    1 мастер-узла — принимает запросы на запись
    
    2 реплик — синхронизируют данные с мастером, обеспечивают отказоустойчивость

  Принцип работы

    Распределение данных — данные автоматически распределяются по шардам на основе хеш-функции (CRC16)
    
    Отказоустойчивость — если мастер-узел падает, одна из реплик автоматически становится мастером
    
    Доступность — даже при падении целой шарды, данные остаются доступны через реплики
    
    Масштабирование — можно добавлять новые шарды без остановки работы

  Конфигурация для развертывания

    version: '3.8'

    services:
      # Шард 1
      redis-master-1:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --appendonly yes
        volumes:
          - data-master-1:/data
        networks:
          - redis-cluster
    
      redis-replica-1-1:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --replicaof redis-master-1 6379
        networks:
          - redis-cluster
    
      redis-replica-1-2:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --replicaof redis-master-1 6379
        networks:
          - redis-cluster
    
      # Шард 2
      redis-master-2:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --appendonly yes
        volumes:
          - data-master-2:/data
        networks:
          - redis-cluster
    
      redis-replica-2-1:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --replicaof redis-master-2 6379
        networks:
          - redis-cluster
    
      redis-replica-2-2:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --replicaof redis-master-2 6379
        networks:
          - redis-cluster
    
      # Шард 3
      redis-master-3:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --appendonly yes
        volumes:
          - data-master-3:/data
        networks:
          - redis-cluster
    
      redis-replica-3-1:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --replicaof redis-master-3 6379
        networks:
          - redis-cluster
    
      redis-replica-3-2:
        image: redis:7.0-alpine
        command: redis-server --cluster-enabled yes --replicaof redis-master-3 6379
        networks:
          - redis-cluster
    
      # Инициализация кластера
      redis-cluster-init:
        image: redis:7.0-alpine
        depends_on:
          - redis-master-1
          - redis-master-2
          - redis-master-3
          - redis-replica-1-1
          - redis-replica-1-2
          - redis-replica-2-1
          - redis-replica-2-2
          - redis-replica-3-1
          - redis-replica-3-2
        command: >
          sh -c "sleep 15 && 
                 redis-cli --cluster create 
                   redis-master-1:6379 
                   redis-master-2:6379 
                   redis-master-3:6379
                   redis-replica-1-1:6379 
                   redis-replica-1-2:6379
                   redis-replica-2-1:6379 
                   redis-replica-2-2:6379
                   redis-replica-3-1:6379 
                   redis-replica-3-2:6379
                   --cluster-replicas 2 
                   --cluster-yes"
        networks:
          - redis-cluster
    
    volumes:
      data-master-1:
      data-master-2:
      data-master-3:
    
    networks:
      redis-cluster:
        driver: bridge
Особенности развертывания
|  Параметр  |	Значение  |	Назначение  |
|  --cluster-enabled yes  |	Включение кластерного режима  |	Активирует распределённое хранение
|  --cluster-replicas 2   |	2 реплики на мастер  |	Обеспечивает отказоустойчивость
|  --appendonly yes  | 	Включение AOF-лога  |	Сохранение данных на диск
|  sleep 15  |	Ожидание запуска узлов  |	Даёт время всем узлам стартовать
