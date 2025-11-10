# Лабораторная работа №2

Работу выполнили:

* Сафьянчиков Арсений Сергеевич
* Жуков Ростислав Сергеевич

---

## Поставленные задачи

1. **Написать “плохой” Dockerfile, в котором есть не менее трех “bad practices” по написанию докерфайлов**
2. **Написать “хороший” Dockerfile, в котором эти плохие практики исправлены**
3. **В Readme описать каждую из плохих практик в плохом докерфайле, почему она плохая и как в хорошем она была исправлена, как исправление повлияло на результат**
4. **В Readme описать 2 плохих практики по работе с контейнерами. ! Не по написанию докерфайлов, а о том, как даже используя хороший докерфайл можно накосячить именно в работе с контейнерами.**

## Определение "Плохого" Docker файла

\- файл, который нарушает стандартные рекомендации по созданию Docker образов. Также такие файлы могут привести к множественным проблемам, таким как безопасность и производительность.

## Ход выполнения работы

Создание "Плохого" Dockerfile с ошибками

Создание "Хорошего" Dokerfile с исправленными ошибками

Сборка и сравнение образов

Создание App.py(+ requirements.txt) для демонстрации работы контейнеров

Описать 2 плохие практики

### Шаг №1: Создание app.py для демонстрации работы

```python
from flask import Flask
import numpy as np

app = Flask(__name__)

@app.route('/')
def hello():
    array = np.array([1, 2, 3, 4, 5])
    mean = np.mean(array)
    return f'Docker Lab Working! Array mean: {mean}'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

Импортируем библиотеки, которые установим в докерфайле 

### Шаг №2: Создание "плохого" Dockerfile

```dockerfile
# ПЛОХОЙ Dockerfile с bad practices
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN pip3 install --break-system-packages flask
RUN pip3 install --break-system-packages requests
RUN pip3 install --break-system-packages numpy

COPY . /app
WORKDIR /app

RUN useradd -m myuser
USER myuser

EXPOSE 5000 // порт 5000

CMD ["python3", "app.py"]
```

**Объяснение ошибок:**

1. `latest` - непредсказуемая версия, может меняться со временем
2. `RUN` - присутствует 6 отдельных `RUN`, то есть 6 слоев. Происходит увеличение образа.
3. `COPY` - копирует все файлы из текущей директории, включая .git, что увеличивает размер образа и создает риск попадания ненужной информации
4. Нарушена методолгия создания файла - создавать юзера нужно до выполнения комманд - `USER`

**Создадим .dockerignore для плохого dockerfile**

```
.git
.gitignore
README.md
Dockerfile
.dockerignore
__pycache__
*.pyc
*.pyo
.env
*.log
```

Да, в самом докерфайле у нас есть `COPY` - как ошибка копирования всех файлов, но тут мы создаем решение этой проблемы. Возможно присутствует противоречие, но я думаю так все таки правильнее.

**Сборка плохого Docker образа** 

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker build -t my-bad-app -f Dockerfile.bad .
[+] Building 247.5s (15/15) FINISHED                                                     docker:desktop-linux
 => [internal] load build definition from Dockerfile.bad                                                 0.0s
 => => transferring dockerfile: 435B                                                                     0.0s
 => [internal] load metadata for docker.io/library/ubuntu:latest                                         2.9s
 => [internal] load .dockerignore                                                                        0.0s
 => => transferring context: 125B                                                                        0.0s
 => [ 1/10] FROM docker.io/library/ubuntu:latest@sha256:66460d557b25769b102175144d538d88219c077c678a49a  0.0s
 => => resolve docker.io/library/ubuntu:latest@sha256:66460d557b25769b102175144d538d88219c077c678a49af4  0.0s
 => [internal] load build context                                                                        0.0s
 => => transferring context: 4.96kB                                                                      0.0s
 => [ 2/10] RUN apt-get update                                                                          12.2s
 => [ 3/10] RUN apt-get install -y python3                                                              12.7s
 => [ 4/10] RUN apt-get install -y python3-pip                                                         136.7s 
 => [ 5/10] RUN pip3 install --break-system-packages flask                                               4.8s 
 => [ 6/10] RUN pip3 install --break-system-packages requests                                            4.2s 
 => [ 7/10] RUN pip3 install --break-system-packages numpy                                              50.4s 
 => [ 8/10] COPY . /app                                                                                  0.1s 
 => [ 9/10] WORKDIR /app                                                                                 0.0s 
 => [10/10] RUN useradd -m myuser                                                                        0.2s 
 => exporting to image                                                                                  23.2s 
 => => exporting layers                                                                                 15.6s 
 => => exporting manifest sha256:a0e5fa428332080b3652782cde63117847705688abe2dee40b96e8bb08012a06        0.0s 
 => => exporting config sha256:56dce59588f72dd3963d8cc5e40cc273a7c039ffe3a495950ad03b39383823db          0.0s
 => => exporting attestation manifest sha256:3abc69ca601b826d4a8731bfc483a3dd33e57c205e170e7d098e944377  0.0s
 => => exporting manifest list sha256:fb9cba3fa832d01d4d7506b837045b9d25ce93f31f3551e97309d4ccac769ac3   0.0s
 => => naming to docker.io/library/my-bad-app:latest                                                     0.0s
 => => unpacking to docker.io/library/my-bad-app:latest                                                  7.6s

View build details: docker-desktop://dashboard/build/desktop-linux/desktop-linux/hj4e58l6ax433k95mffuzmvnd
```

**Размер**: 

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker images | grep my-bad-app
my-bad-app    latest    fb9cba3fa832   2 minutes ago   950MB
```

**Запуск:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker run -d -p 5005:5000 --name bad-container my-bad-app
41618c72b4aaf0644a24afaf4745b0ed89030f35ab1ac42dbbe078e3df60158c
```

**Проверка работы:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % curl http://localhost:5005
Docker Lab Working! Array mean: 3.0%
```

**Статус контейнера:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker ps
CONTAINER ID   IMAGE        COMMAND            CREATED         STATUS         PORTS                    NAMES
41618c72b4aa   my-bad-app   "python3 app.py"   9 minutes ago   Up 9 minutes   0.0.0.0:5005->5000/tcp   bad-container
```



### Шаг №3: Создание "Хорошего" Dockerfile

```dockerfile
# ХОРОШИЙ Dockerfile с исправлениями
FROM python:3.9-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

USER appuser

EXPOSE 5000

CMD ["python3", "app.py"]
```

**Исправленные ошибки:**

1. `FROM python:3.9-slim` - фиксированная версия с минимальным размером
2. Создание пользователя в начале файла. `RUN groupadd -r appuser && useradd -r -g appuser appuser` создается системный юзер, без home директории с одним слоем
3. Копирование зависимостей через requirements.txt с одним слоем

**Сборка хорошего Docker образа:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker build -t my-good-app -f Dockerfile.good .
[+] Building 102.5s (11/11) FINISHED                                                     docker:desktop-linux
 => [internal] load build definition from Dockerfile.good                                                0.0s
 => => transferring dockerfile: 335B                                                                     0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                       3.6s
 => [internal] load .dockerignore                                                                        0.0s
 => => transferring context: 125B                                                                        0.0s
 => [1/6] FROM docker.io/library/python:3.9-slim@sha256:2d97f6910b16bd338d3060f261f53f144965f755599aab  19.8s
 => => resolve docker.io/library/python:3.9-slim@sha256:2d97f6910b16bd338d3060f261f53f144965f755599aab1  0.0s
 => => sha256:ea56f685404adf81680322f152d2cfec62115b30dda481c2c450078315beb508 251B / 251B               0.4s
 => => sha256:b3ec39b36ae8c03a3e09854de4ec4aa08381dfed84a9daa075048c2e3df3881d 1.29MB / 1.29MB           2.5s
 => => sha256:fc74430849022d13b0d44b8969a953f842f59c6e9d1a0c2c83d710affa286c08 13.88MB / 13.88MB         5.6s
 => => sha256:38513bd7256313495cdd83b3b0915a633cfa475dc2a07072ab2c8d191020ca5d 29.78MB / 29.78MB        16.6s
 => => extracting sha256:38513bd7256313495cdd83b3b0915a633cfa475dc2a07072ab2c8d191020ca5d                2.0s
 => => extracting sha256:b3ec39b36ae8c03a3e09854de4ec4aa08381dfed84a9daa075048c2e3df3881d                0.1s
 => => extracting sha256:fc74430849022d13b0d44b8969a953f842f59c6e9d1a0c2c83d710affa286c08                1.0s
 => => extracting sha256:ea56f685404adf81680322f152d2cfec62115b30dda481c2c450078315beb508                0.0s
 => [internal] load build context                                                                        0.0s
 => => transferring context: 8.77kB                                                                      0.0s
 => [2/6] RUN groupadd -r appuser && useradd -r -g appuser appuser                                       0.6s
 => [3/6] WORKDIR /app                                                                                   0.0s
 => [4/6] COPY requirements.txt ./                                                                       0.0s
 => [5/6] RUN pip install --no-cache-dir -r requirements.txt                                            74.3s
 => [6/6] COPY . .                                                                                       0.0s
 => exporting to image                                                                                   4.0s
 => => exporting layers                                                                                  3.0s
 => => exporting manifest sha256:27725e836a73e160e625547505d77d9631adc6bab66c08d1d05ee7b68f8c7a27        0.0s
 => => exporting config sha256:5b004e5fafbc3ffc5e405e21a390e22360a07d1df4816f3b1c8da63f7e71ef85          0.0s
 => => exporting attestation manifest sha256:41df91fd14bac13027efd336526b7048d0b7733ce9fadb0b0b14bd4aa0  0.0s
 => => exporting manifest list sha256:7b6e303ff2e6d789326b6a137431e3da37e690699e931dc5abc8f7871569ef40   0.0s
 => => naming to docker.io/library/my-good-app:latest                                                    0.0s
 => => unpacking to docker.io/library/my-good-app:latest                                                 0.9s

View build details: docker-desktop://dashboard/build/desktop-linux/desktop-linux/xnl3qg8aiii3emienwigc41eh
```

**Размер**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker images | grep my-good-app
my-good-app   latest    7b6e303ff2e6   3 minutes ago    296MB
```

**Запуск:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker run -d -p 5006:5000 --name good-container my-good-app
21e275457d3f87c86ffddabc26e681dc7f7ecbb598add20b05e927d34ada8801
```

**Проверка работы:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % curl http://localhost:5006
Docker Lab Working! Array mean: 3.0% 
```

**Статус контейнера:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker ps | grep good-container
21e275457d3f   my-good-app   "python3 app.py"   2 minutes ago    Up 2 minutes    0.0.0.0:5006->5000/tcp   good-container
```


### Сравнение "плохого" и "хорошего" Dockerfile

**Сравнение размеров:**

```bash
arsenijsafancikov@MacBook-Pro-4 lb2 % docker images | grep my-
my-good-app   latest    7b6e303ff2e6   11 minutes ago   296MB
my-bad-app    latest    fb9cba3fa832   50 minutes ago   950MB
```

**Сравнение:**

Плохой образ:

1. 13 слоев
2. 950MB размер
3. Тяжелые слои: `apt-get install python3-pip` - 401MB; `pip3 install numpy` - 90.5MB

Хороший образ:

1. 9 слоев
2. 539MB размер
3. Тяжелые слои: `pip install -r requirements.txt` - 88MB; 


### Описание 2-х плохих практик

**1. Хранение данных внутри контейнера**

Данные теряются при пересоздании контейнера

Неправильно:

```bash
docker run my-app
```

\- приложение пишет данные в /app/data внутри контейнера

при удалении контейнера - данные теряются 

Правльно:

```bash
docker volume create app_data
docker run -v app_data:/app/data my-app
```

**2. Запускмножества процессов в одном контейнере**

Сложность маштабирования, противоречие принципу "single responsibility"

Неправильно:

```bash
docker run -d mysupercontainer
```

Правильно:

```bash
docker run -d --name database postgres
docker run -d --name cache redis  
docker run -d --name webapp my-app
```

\- Разделение на специализированные контейнеры

## Вывод

В ходе выполнения лабораторной работы, мы узнали как правильно составлять Dockerfiles, узнали рекомендации по составлению и поняли как делать не нужно. Мы составили плохой Dockerfile с ошибками и исправили эти недочеты в хорошем Dockerfile.
