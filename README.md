# Spring Cloud Config — Домашнее задание

Цель: поднять **Config Server** и **Config Client**, хранить конфигурации в **отдельном Git-репозитории**, получить значение параметра из конфига по HTTP.

---

## Репозитории

- Домашка (этот): https://github.com/Luchente/spring-cloud-config-hw
- Репозиторий конфигураций: https://github.com/Luchente/spring-cloud-config-repo

> В конфиг-репозитории лежат файлы:
> - `client-app.yaml` — дефолт
> - `client-app-dev.yaml` — профиль `dev`
> - `client-app-prod.yaml` — профиль `prod`
> - (опц.) `application.yaml` — общие настройки

---

## Стек и версии (проверенные)

- **Java:** 17 (рекомендовано). Можно 21, если Maven запускается на JDK 21.
- **Spring Boot:** 3.4.11
- **Spring Cloud:** 2024.0.0 (совместим с Boot 3.4.x)
- **Сборка:** Maven Wrapper (`./mvnw`, `mvnw.cmd`)

---

## Структура проекта

~~~text
spring-cloud-config-hw/
  README.md
  config-server/
    pom.xml
    src/main/java/.../ConfigServerApplication.java
    src/main/resources/application.yml
    .mvn/wrapper/...
    mvnw, mvnw.cmd
  config-client/
    pom.xml
    src/main/java/.../ConfigClientApplication.java
    src/main/java/.../MessageController.java
    src/main/resources/application.yml
    .mvn/wrapper/...
    mvnw, mvnw.cmd
~~~

---

## Как это работает (коротко)

1. **Config Server** читает YAML из Git-репозитория.
2. **Config Client** при старте идёт на сервер по `spring.config.import=configserver:...`,
   представляется как `spring.application.name=client-app`, указывает профиль (`dev`/`prod`)
   и получает параметры.
3. Контроллер клиента отдаёт параметр `app.message` по `GET /message`.

---

## Быстрый старт

### 1) Config Server (порт 8888)

~~~bash
cd config-server
./mvnw spring-boot:run
~~~

Проверки:

~~~bash
# здоровье
curl http://localhost:8888/actuator/health

# конфиги для приложения client-app под разными профилями
curl http://localhost:8888/client-app/dev
curl http://localhost:8888/client-app/prod
~~~

Ожидается JSON, где видно `app.message: Hello from DEV/PROD`.

---

### 2) Config Client (порт 8080)

~~~bash
cd config-client

# DEV
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
curl http://localhost:8080/message         # -> Hello from DEV

# Остановите процесс (Ctrl+C), затем PROD:
./mvnw spring-boot:run -Dspring-boot.run.profiles=prod
curl http://localhost:8080/message         # -> Hello from PROD
~~~

---

## Конфигурация — важные файлы

### `config-server/src/main/resources/application.yml`
~~~yaml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/Luchente/spring-cloud-config-repo.git
          default-label: main
          clone-on-start: true

management:
  endpoints:
    web:
      exposure:
        include: health,info
~~~

### `config-client/src/main/resources/application.yml`
~~~yaml
server:
  port: 8080

spring:
  application:
    name: client-app
  config:
    import: "optional:configserver:http://localhost:8888"
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        multiplier: 1.5
        max-interval: 5000

management:
  endpoints:
    web:
      exposure:
        include: health,info,refresh
~~~

### Контроллер клиента
~~~java
// src/main/java/.../MessageController.java
package ru.netology.configclient;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RefreshScope
@RestController
public class MessageController {

  @Value("${app.message:UNDEFINED}")
  private String message;

  @GetMapping("/message")
  public String message() {
    return message;
  }
}
~~~

---

## (Опционально) Горячее обновление без рестарта

1) Измените в конфиг-репозитории `client-app-dev.yaml` (например, `Hello from DEV v2`), закоммитьте и запушьте.
2) На клиенте (под профилем `dev`) выполните:

~~~bash
curl -X POST http://localhost:8080/actuator/refresh
curl http://localhost:8080/message
# -> увидите обновлённое значение
~~~

---

## Трюки и полезности

- **Запуск через JAR:**
  ~~~bash
  # server
  cd config-server && ./mvnw -DskipTests package
  java -jar target/config-server-0.0.1-SNAPSHOT.jar

  # client (dev)
  cd ../config-client && ./mvnw -DskipTests package
  java -Dspring.profiles.active=dev -jar target/config-client-0.0.1-SNAPSHOT.jar
  ~
