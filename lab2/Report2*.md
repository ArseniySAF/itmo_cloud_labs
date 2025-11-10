# Лабораторная работа №2*

Работу выполнили:

* Сафьянчиков Арсений Сергеевич
* Жуков Ростислав Сергеевич

---

## Поставленные задачи

1. **Написать “плохой” Docker compose файл, в котором есть не менее трех “bad practices” по их написанию**
2. **Написать “хороший” Docker compose файл, в котором эти плохие практики исправлены**
3. **В Readme описать каждую из плохих практик в плохом файле, почему она плохая и как в хорошем она была исправлена, как исправление повлияло на результат**
4. **После предыдущих пунктов в хорошем файле настроить сервисы так, чтобы контейнеры в рамках этого compose-проекта так же поднимались вместе, но не "видели" друг друга по сети. В отчете описать, как этого добились и кратко объяснить принцип такой изоляции**

## Ход выполнения работы

Создание плохого docker compose файла

Создание хорошего docker compose файла

Настройка сетевой изоляции

### Шаг №1: Создание "Плохого" docker compose файла

```yaml
version: '3'

services:
  web:
    image: python:latest
    command: python3 -c "import time; print('Web service started'); time.sleep(3600)"
    ports:
      - "5000:5000"
    volumes:
      - .:/app
    depends_on:
      - db

  db:
    image: postgres:latest
    environment:
      POSTGRES_PASSWORD: password123
    volumes:
      - /var/lib/postgresql/data

  cache:
    image: redis:latest
```

**Объяснение ошибок:**

1. `latest` - непредсказуемая версия, новая версия может сломать программу
2. `POSTGRES_PASSWORD` - пароль в коде может привлечь на собой уязвимость и утечки + при смене пароля нужно будет копаться в коде и искать его
3. Отсутствие restart policy - контейнеры не перезапусскаются при падении
4. Присутствует anonymous volume - не имеет имени неявное поведение при сбое
5. Отсутствует изоляция - все видят друг друга + высокая уязвимость
6. (не ошибка) Используем `command: python3....`, а не `app.py`, так как делаем акцент не на импорте зависимостей, а на видимости, сетях и тд

**Сборка и запуск плохого docker compose файла:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker-compose -f docker-compose.bad.yml up -d
WARN[0000] /Users/arsenijsafancikov/University/DevOps/lb2*/docker-compose.bad.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 3/3
 ✔ Container lb2-db-1     Started                                                                        0.5s 
 ✔ Container lb2-cache-1  Running                                                                        0.0s 
 ✔ Container lb2-web-1    Started   
```

**Проверка статуса:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker-compose -f docker-compose.bad.yml ps
WARN[0000] /Users/arsenijsafancikov/University/DevOps/lb2*/docker-compose.bad.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
NAME          IMAGE          COMMAND                  SERVICE   CREATED         STATUS         PORTS
lb2-cache-1   redis:latest   "docker-entrypoint.s…"   cache     2 minutes ago   Up 2 minutes   6379/tcp
```

**Проверка сетей:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker network ls
NETWORK ID     NAME          DRIVER    SCOPE
2761368877a9   bridge        bridge    local
fd70f34ab739   host          host      local
b09e77d742d5   lb2_default   bridge    local
cdd387baf50a   none          null      local
```

Видим, что создалась что создалась default папка

**Размеры:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker images | grep -E "(python|postgres|redis)"

python        latest    1ad1a43b5e24   5 days ago    1.61GB
postgres      latest    41fc5342eefb   5 days ago    643MB
redis         latest    5c7c0445ed86   5 days ago    200MB
```


### Шаг №2: Создаем "Хорошего" docker compose файла

```yaml
services:
  web:
    image: python:3.9-slim
    command: python3 -c "import time; print('Web service started'); time.sleep(3600)"
    ports:
      - "5008:5000"
    environment:
      - FLASK_ENV=production
    depends_on:
      - db
    restart: unless-stopped
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: myapp
      POSTGRES_USER: app_user
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - app-network

  cache:
    image: redis:7-alpine
    restart: unless-stopped
    networks:
      - app-network

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
```

**Исправленные ошибки:**

1. Фиксированные версии вместо `latest` - `image: python:3.9-slim`. Предсказуемость поведения
2. `NAMED VOLUMES` вместо `ANONYMOUS`. `postgres_data:/.....`. Сохранение данных при docker-compose down, управляемость с помощью docker volume ls.
3. Добавлена restart policy. `restart: unless-stopped`. Автоматическое восстановление при падении.
4. Добавлены явные сети, что дает изоляцию от других проектов и безопасность

   ```
   networks:
     - app-network

   networks:
     app-network:
       driver: bridge
   ```
5. Также создан дополнительный файл для хранения пароля - `db_password.txt`. Это предотвращает утечки паролей. `POSTGRES_PASSWORD_FILE: /run/secrets/db_password`

**Сборка и запуск плохого docker compose файла:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker-compose -f docker-compose.good.yml up -d
[+] Running 3/3
 ✔ Container lb2-db-1     Started                                                                        1.2s 
 ✔ Container lb2-cache-1  Started                                                                        1.2s 
 ✔ Container lb2-web-1    Started
```

**Проверка статуса:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker-compose -f docker-compose.good.yml ps
NAME          IMAGE                COMMAND                  SERVICE   CREATED              STATUS                         PORTS
lb2-cache-1   redis:7-alpine       "docker-entrypoint.s…"   cache     About a minute ago   Up About a minute              6379/tcp
lb2-db-1      postgres:15-alpine   "docker-entrypoint.s…"   db        About a minute ago   Restarting (1) 6 seconds ago   
lb2-web-1     python:3.9-slim      "python3 -c 'import …"   web       About a minute ago   Up About a minute              0.0.0.0:5008->5000/tcp
```

**Размеры:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker images | grep -E "(python|postgres|redis)"
python        latest      1ad1a43b5e24   5 days ago     1.61GB
postgres      latest      41fc5342eefb   6 days ago     643MB
redis         latest      5c7c0445ed86   6 days ago     200MB
redis         7-alpine    ee64a64eaab6   6 days ago     60.7MB
python        3.9-slim    2d97f6910b16   9 days ago     183MB
postgres      15-alpine   64583b3cb4f2   3 weeks ago    390MB
```

**Проверка сетей:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker network ls | grep lb2
298b3be5903f   lb2_app-network   bridge    local
9cb9d7a49c3d   lb2_default       bridge    local
```


### Шаг №3: Создание изолированного compose файла

```yaml
version: '3.8'

services:
  web:
    image: python:3.9-slim
    command: python3 -c "import time; print('Web service started'); time.sleep(3600)"
    ports:
      - "5009:5000"
    networks:
      - web-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_PASSWORD: s123
      POSTGRES_DB: myapp
    networks:
      - db-network

  cache:
    image: redis:7-alpine
    networks:
      - cache-network

networks:
  web-network:
    driver: bridge
  db-network:
    driver: bridge
  cache-network:
    driver: bridge
```

**Запускаем изолированыый compose:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker-compose -f docker-compose.isolated.yml down
docker-compose -f docker-compose.isolated.yml up -d
[+] Running 6/6
 ✔ Network lb2_web-network    Created                                                                    0.1s 
 ✔ Network lb2_db-network     Created                                                                    0.1s 
 ✔ Network lb2_cache-network  Created                                                                    0.1s 
 ✔ Container lb2-db-1         Started                                                                    0.6s 
 ✔ Container lb2-web-1        Started                                                                    0.7s 
 ✔ Container lb2-cache-1      Started
```

**Проверяем изоляцию compose:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker-compose -f docker-compose.isolated.yml up -d
docker-compose -f docker-compose.isolated.yml exec web python3 -c "
import socket
try:
    ip = socket.gethostbyname('db')
    print(f'SUCCESS: db resolves to {ip}')
except Exception as e:
    print(f'FAILED: {e}')
"
WARN[0000] /Users/arsenijsafancikov/University/DevOps/lb2*/docker-compose.isolated.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
[+] Running 3/3
 ✔ Container lb2-db-1     Started                                                                       10.8s 
 ✔ Container lb2-web-1    Started                                                                       10.8s 
 ✔ Container lb2-cache-1  Started                                                                       10.7s 
WARN[0000] /Users/arsenijsafancikov/University/DevOps/lb2*/docker-compose.isolated.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion 
FAILED: [Errno -2] Name or service not known
```

Контейнер web не может найти имя db - сервисы изолированы в разных сетях

**Сравниваем с хорошим compose:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2* % docker-compose -f docker-compose.good.yml up -d
docker-compose -f docker-compose.good.yml exec web python3 -c "
import socket
try:
    ip = socket.gethostbyname('db')
    print(f'SUCCESS: db resolves to {ip}')
except Exception as e:
    print(f'FAILED: {e}')
"
[+] Running 4/4
 ✔ Network lb2_app-network  Created                                                                      0.1s 
 ✔ Container lb2-cache-1    Started                                                                     11.3s 
 ✔ Container lb2-db-1       Started                                                                     11.3s 
 ✔ Container lb2-web-1      Started                                                                     11.0s 
SUCCESS: db resolves to 172.21.0.4
```

Тут все работает.


## Вывод

В ходе выполнения лабораторной работы мы узнали, что такое контейнерная оркестрация с помощью Docker Compose. Написали плохой Docker compose и исправили их в хорошем Docker compose. Также настроили изоляцию.  Теперь возможно запускать связанные сервисы в изолированнных сетях.
