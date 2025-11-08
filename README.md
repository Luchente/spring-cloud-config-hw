# Spring Cloud Config — HW

## Репозитории
- Домашка (этот): https://github.com/Luchente/spring-cloud-config-hw
- Репозиторий конфигураций: https://github.com/Luchente/spring-cloud-config-repo

## Запуск

### 1) Config Server (порт 8888)
```bash
cd config-server
./mvnw spring-boot:run
# Проверка:
curl http://localhost:8888/actuator/health
curl http://localhost:8888/client-app/dev
curl http://localhost:8888/client-app/prod
