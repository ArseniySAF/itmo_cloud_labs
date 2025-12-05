# Лабораторная работа №3

Работу выполнили:

- Сафьянчиков Арсений Сергеевич
- Жуков Ростислав Сергеевич

---

## Поставленные задачи

1. **Написать “плохой” CI/CD файл, который работает, но в нем есть не менее трех “bad practices” по написанию CI/CD**
2. **Написать “хороший” CI/CD, в котором эти плохие практики исправлены**
3. **В Readme описать каждую из плохих практик в плохом файле, почему она плохая и как в хорошем она была исправлена, как исправление повлияло на результат**
4. **Прочитать историю про Васю))**
5. **Написать обычный отчет**

## Определение "Плохого" CI/CD файла

**Плохой CI/CD файл** — это конфигурация конвейера, которая формально выполняет свои задачи, но построена с нарушением лучших практик: содержит небезопасные или нестабильные решения, плохо масштабируется, усложняет поддержку и может приводить к ошибкам в процессе разработки или деплоя.

## Ход выполнения работы

Создание архитектуры

Создание "Плохого" CI/CD файла

Создание "Хорошего" CI/CD файла с исправленными ошибками

Сравнить и описать хороший и плохой ci

Прочитать статью про Васю

### Шаг №1: Создание архитектуры

**App.py** - функция/пример для теста

```python
def add(a: int, b: int) -> int:
    return a + b


if __name__ == "__main__":
    x = int(input("first number: "))
    y = int(input("second number: "))
    print(add(x, y))

```

**requirements.txt** - для зависимостей + блок версии

**tests/test_app.py** - сам тест

```python
from app import add


def test_add_positive():
    assert add(2, 3) == 5


def test_add_zero():
    assert add(0, 0) == 0

```

**.github/workflows/bad-ci.yml и .github/workflows/good-ci.yml** - сами yaml файлы. На гите все файлы, описывающие worlflow должны находиться на таком пути, иначе они будут проигнорированы.

### Шаг №2: Создание "плохого" CI/CD файла

```yaml
name: Bad CI

on:
  push:
    branches:
      - '*'
  pull_request:

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest

      - name: Run tests
        run: |
          echo "SUPER_SECRET = ${{ secrets.SUPER_SECRET }}"
          pytest

      - name: Deploy to production
        if: github.ref == 'refs/heads/main'
        run: |
          echo "Deploying to PRODUCTION from branch ${GITHUB_REF}"
          ssh user@prod-server "cd /app && git pull && systemctl restart app"

```

**Bad pactice 1: Логирование секрета в консоль**

```yaml
echo "SUPER_SECRET = ${{ secrets.SUPER_SECRET }}"

```

Логи попадают в гит и могут быть видны третьим лицам.

Нарушение практики работы

**Bad practice 2: Деплой/развертывание приложения без защит**

```yaml
- name: Deploy to production
  if: github.ref == 'refs/heads/main'
  run: |
    ssh user@prod-server "cd /app && git pull && systemctl restart app"

```

Деплой происходит сразу при любом пуше в main и если в нем будет ошибка -> проект упадет

Нет защиты ревью/тестов - плохая практика.

**Bad practice 3: нет статичной версии pyhon**

```yaml
uses: actions/setup-python@v5
with:
  python-version: '3.x'

```

Версия Python задана `3.x` - плохо. Возможны различные поведения кода.

Также можно заметить что action имеет указанную 5 версию - тоже может вызвать проблемы, но вообще норм.

### Шаг №3: Создание "хорошего" CI/CD файла

```yaml
name: Good CI/CD

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
      - develop
  workflow_dispatch:

jobs:
  tests:
    name: Run tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests
        run: python -m pytest

  deploy:
    name: Deploy to production
    needs: tests
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    environment:
      name: production
      url: https://example.com/app

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Deploy using script
        run: |
          echo "Starting deployment to production..."

```

**Исправление 1**

нет логирование секретов через echo

```yaml
environment:
  name: production

```

Секреты не утекают. Разработка становиться безопаной

**Исправление 2**

Сейчас деплой разделен на 2 этапа: test и deploy

```yaml
needs: tests

```

Деплой на прод возможен только через успешные тесты

также деплой будет запускаться только на push в main

```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'

```

**Исправление 3**

Сделали версию python конкретной

```yaml
python-version: '3.12'
```

### Шаг №4: Заупустим CI/CD локально через Act для проверки.

Инициализируем git

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main

```

запускаем CI через act

```bash
act push -W .github/workflows/good-ci.yml

```

Выбираем Medium image

и при первом запуске получаем ошибку:

```bash
==================================== ERRORS ====================================
| ______________________ ERROR collecting tests/test_app.py ______________________
| ImportError while importing test module '/Users/arsenijsafancikov/University/DevOps/lb3/tests/test_app.py'.
| Hint: make sure your test modules/packages have valid Python names.
| Traceback:
| /opt/hostedtoolcache/Python/3.12.12/x64/lib/python3.12/importlib/__init__.py:90: in import_module
|     return _bootstrap._gcd_import(name[level:], package, level)
| tests/test_app.py:1: in <module>
|     from app import add
| E   ModuleNotFoundError: No module named 'app'
| =========================== short test summary info ============================
| ERROR tests/test_app.py
| !!!!!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!!!!!
| =============================== 1 error in 0.11s ===============================
```

Исправляем. нужно было указать путь для app

Успешные логи:

```bash
arsenijsafancikov@MacBook-Pro-4 lb3 % act push -W .github/workflows/good-ci.yml

INFO[0000] Using docker host 'unix:///var/run/docker.sock', and daemon socket 'unix:///var/run/docker.sock' 
[Good CI/CD/Run tests] ⭐ Run Set up job
[Good CI/CD/Run tests] 🚀  Start image=catthehacker/ubuntu:act-latest
[Good CI/CD/Run tests]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform= username= forcePull=true
[Good CI/CD/Run tests]   🐳  docker create image=catthehacker/ubuntu:act-latest platform= entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Good CI/CD/Run tests]   🐳  docker run image=catthehacker/ubuntu:act-latest platform= entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Good CI/CD/Run tests]   🐳  docker exec cmd=[node --no-warnings -e console.log(process.execPath)] user= workdir=
[Good CI/CD/Run tests]   ✅  Success - Set up job
[Good CI/CD/Run tests]   ☁  git clone 'https://github.com/actions/setup-python' # ref=v5
[Good CI/CD/Run tests]   ☁  git clone 'https://github.com/actions/cache' # ref=v4
[Good CI/CD/Run tests] ⭐ Run Main Checkout code
[Good CI/CD/Run tests]   🐳  docker cp src=/Users/arsenijsafancikov/University/DevOps/lb3/. dst=/Users/arsenijsafancikov/University/DevOps/lb3
[Good CI/CD/Run tests]   ✅  Success - Main Checkout code [55.053886ms]
[Good CI/CD/Run tests] ⭐ Run Main Setup Python
[Good CI/CD/Run tests]   🐳  docker cp src=/Users/arsenijsafancikov/.cache/act/actions-setup-python@v5/ dst=/var/run/act/actions/actions-setup-python@v5/
[Good CI/CD/Run tests]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/actions/actions-setup-python@v5/dist/setup/index.js] user= workdir=
[Good CI/CD/Run tests]   ❓  ::group::Installed versions
| Successfully set up CPython (3.12.12)
[Good CI/CD/Run tests]   ❓  ::endgroup::
[Good CI/CD/Run tests]   ❓ add-matcher /run/act/actions/actions-setup-python@v5/.github/python.json
[Good CI/CD/Run tests]   ✅  Success - Main Setup Python [1.40061433s]
[Good CI/CD/Run tests]   ⚙  ::set-env:: LD_LIBRARY_PATH=/opt/hostedtoolcache/Python/3.12.12/x64/lib
[Good CI/CD/Run tests]   ⚙  ::set-env:: pythonLocation=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Run tests]   ⚙  ::set-env:: PKG_CONFIG_PATH=/opt/hostedtoolcache/Python/3.12.12/x64/lib/pkgconfig
[Good CI/CD/Run tests]   ⚙  ::set-env:: Python_ROOT_DIR=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Run tests]   ⚙  ::set-env:: Python2_ROOT_DIR=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Run tests]   ⚙  ::set-env:: Python3_ROOT_DIR=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Run tests]   ⚙  ::set-output:: python-version=3.12.12
[Good CI/CD/Run tests]   ⚙  ::set-output:: python-path=/opt/hostedtoolcache/Python/3.12.12/x64/bin/python
[Good CI/CD/Run tests]   ⚙  ::add-path:: /opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Run tests]   ⚙  ::add-path:: /opt/hostedtoolcache/Python/3.12.12/x64/bin
[Good CI/CD/Run tests]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/workflow/hashfiles/index.js] user= workdir=
[Good CI/CD/Run tests] ⭐ Run Main Cache pip
[Good CI/CD/Run tests]   🐳  docker cp src=/Users/arsenijsafancikov/.cache/act/actions-cache@v4/ dst=/var/run/act/actions/actions-cache@v4/
[Good CI/CD/Run tests]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/actions/actions-cache@v4/dist/restore/index.js] user= workdir=
| Cache not found for input keys: Linux-pip-03ce6d207afda3f8e7f5e45e89a3c7a01149af25dee48e1c5f3f6a6b681c88a7, Linux-pip-
[Good CI/CD/Run tests]   ✅  Success - Main Cache pip [1.549643879s]
[Good CI/CD/Run tests] ⭐ Run Main Install dependencies
[Good CI/CD/Run tests]   🐳  docker exec cmd=[bash -e /var/run/act/workflow/3] user= workdir=
| Requirement already satisfied: pip in /opt/hostedtoolcache/Python/3.12.12/x64/lib/python3.12/site-packages (25.3)
| WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
| Requirement already satisfied: pytest==8.3.3 in /opt/hostedtoolcache/Python/3.12.12/x64/lib/python3.12/site-packages (from -r requirements.txt (line 1)) (8.3.3)
| Requirement already satisfied: iniconfig in /opt/hostedtoolcache/Python/3.12.12/x64/lib/python3.12/site-packages (from pytest==8.3.3->-r requirements.txt (line 1)) (2.3.0)
| Requirement already satisfied: packaging in /opt/hostedtoolcache/Python/3.12.12/x64/lib/python3.12/site-packages (from pytest==8.3.3->-r requirements.txt (line 1)) (25.0)
| Requirement already satisfied: pluggy<2,>=1.5 in /opt/hostedtoolcache/Python/3.12.12/x64/lib/python3.12/site-packages (from pytest==8.3.3->-r requirements.txt (line 1)) (1.6.0)
| WARNING: Running pip as the 'root' user can result in broken permissions and conflicting behaviour with the system package manager, possibly rendering your system unusable. It is recommended to use a virtual environment instead: https://pip.pypa.io/warnings/venv. Use the --root-user-action option if you know what you are doing and want to suppress this warning.
[Good CI/CD/Run tests]   ✅  Success - Main Install dependencies [1.339534522s]
[Good CI/CD/Run tests] ⭐ Run Main Run tests
[Good CI/CD/Run tests]   🐳  docker exec cmd=[bash -e /var/run/act/workflow/4] user= workdir=
| ============================= test session starts ==============================
| platform linux -- Python 3.12.12, pytest-8.3.3, pluggy-1.6.0
| rootdir: /Users/arsenijsafancikov/University/DevOps/lb3
collected 2 items                                                        
| 
| tests/test_app.py ..                                                     [100%]
| 
| ============================== 2 passed in 0.01s ===============================
[Good CI/CD/Run tests]   ✅  Success - Main Run tests [392.220888ms]
[Good CI/CD/Run tests]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/workflow/hashfiles/index.js] user= workdir=
[Good CI/CD/Run tests] ⭐ Run Post Cache pip
[Good CI/CD/Run tests]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/actions/actions-cache@v4/dist/save/index.js] user= workdir=
| [command]/usr/bin/tar --posix -cf cache.tzst --exclude cache.tzst -P -C /Users/arsenijsafancikov/University/DevOps/lb3 --files-from manifest.txt --use-compress-program zstdmt
| Cache Size: ~0 MB (38804 B)
| Cache saved successfully
| Cache saved with key: Linux-pip-03ce6d207afda3f8e7f5e45e89a3c7a01149af25dee48e1c5f3f6a6b681c88a7
[Good CI/CD/Run tests]   ✅  Success - Post Cache pip [507.184066ms]
[Good CI/CD/Run tests] ⭐ Run Post Setup Python
[Good CI/CD/Run tests]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/actions/actions-setup-python@v5/dist/cache-save/index.js] user= workdir=
[Good CI/CD/Run tests]   ✅  Success - Post Setup Python [342.746407ms]
[Good CI/CD/Run tests] ⭐ Run Complete job
[Good CI/CD/Run tests] Cleaning up container for job Run tests
[Good CI/CD/Run tests]   ✅  Success - Complete job
[Good CI/CD/Run tests] 🏁  Job succeeded
[Good CI/CD/Deploy to production] ⭐ Run Set up job
[Good CI/CD/Deploy to production] 🚀  Start image=catthehacker/ubuntu:act-latest
[Good CI/CD/Deploy to production]   🐳  docker pull image=catthehacker/ubuntu:act-latest platform= username= forcePull=true
[Good CI/CD/Deploy to production]   🐳  docker create image=catthehacker/ubuntu:act-latest platform= entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Good CI/CD/Deploy to production]   🐳  docker run image=catthehacker/ubuntu:act-latest platform= entrypoint=["tail" "-f" "/dev/null"] cmd=[] network="host"
[Good CI/CD/Deploy to production]   🐳  docker exec cmd=[node --no-warnings -e console.log(process.execPath)] user= workdir=
[Good CI/CD/Deploy to production]   ✅  Success - Set up job
[Good CI/CD/Deploy to production]   ☁  git clone 'https://github.com/actions/setup-python' # ref=v5
[Good CI/CD/Deploy to production] ⭐ Run Main Checkout code
[Good CI/CD/Deploy to production]   🐳  docker cp src=/Users/arsenijsafancikov/University/DevOps/lb3/. dst=/Users/arsenijsafancikov/University/DevOps/lb3
[Good CI/CD/Deploy to production]   ✅  Success - Main Checkout code [47.18411ms]
[Good CI/CD/Deploy to production] ⭐ Run Main Setup Python
[Good CI/CD/Deploy to production]   🐳  docker cp src=/Users/arsenijsafancikov/.cache/act/actions-setup-python@v5/ dst=/var/run/act/actions/actions-setup-python@v5/
[Good CI/CD/Deploy to production]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/actions/actions-setup-python@v5/dist/setup/index.js] user= workdir=
[Good CI/CD/Deploy to production]   ❓  ::group::Installed versions
| Successfully set up CPython (3.12.12)
[Good CI/CD/Deploy to production]   ❓  ::endgroup::
[Good CI/CD/Deploy to production]   ❓ add-matcher /run/act/actions/actions-setup-python@v5/.github/python.json
[Good CI/CD/Deploy to production]   ✅  Success - Main Setup Python [1.464834474s]
[Good CI/CD/Deploy to production]   ⚙  ::set-env:: Python_ROOT_DIR=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Deploy to production]   ⚙  ::set-env:: Python2_ROOT_DIR=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Deploy to production]   ⚙  ::set-env:: Python3_ROOT_DIR=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Deploy to production]   ⚙  ::set-env:: LD_LIBRARY_PATH=/opt/hostedtoolcache/Python/3.12.12/x64/lib
[Good CI/CD/Deploy to production]   ⚙  ::set-env:: pythonLocation=/opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Deploy to production]   ⚙  ::set-env:: PKG_CONFIG_PATH=/opt/hostedtoolcache/Python/3.12.12/x64/lib/pkgconfig
[Good CI/CD/Deploy to production]   ⚙  ::set-output:: python-version=3.12.12
[Good CI/CD/Deploy to production]   ⚙  ::set-output:: python-path=/opt/hostedtoolcache/Python/3.12.12/x64/bin/python
[Good CI/CD/Deploy to production]   ⚙  ::add-path:: /opt/hostedtoolcache/Python/3.12.12/x64
[Good CI/CD/Deploy to production]   ⚙  ::add-path:: /opt/hostedtoolcache/Python/3.12.12/x64/bin
[Good CI/CD/Deploy to production] ⭐ Run Main Deploy using script
[Good CI/CD/Deploy to production]   🐳  docker exec cmd=[bash -e /var/run/act/workflow/2] user= workdir=
| Starting deployment to production...
[Good CI/CD/Deploy to production]   ✅  Success - Main Deploy using script [179.117914ms]
[Good CI/CD/Deploy to production] ⭐ Run Post Setup Python
[Good CI/CD/Deploy to production]   🐳  docker exec cmd=[/opt/acttoolcache/node/18.20.8/x64/bin/node /var/run/act/actions/actions-setup-python@v5/dist/cache-save/index.js] user= workdir=
[Good CI/CD/Deploy to production]   ✅  Success - Post Setup Python [370.247049ms]
[Good CI/CD/Deploy to production] ⭐ Run Complete job
[Good CI/CD/Deploy to production] Cleaning up container for job Deploy to production
[Good CI/CD/Deploy to production]   ✅  Success - Complete job
[Good CI/CD/Deploy to production] 🏁  Job succeeded
```

Видим что Job test успешно выполнилось - прошли тесты

## Вывод

В ходе работы мы настроили плохой и хороший CI/CD, что позволилона практики понять и увидеть какие bad practice существуют и как они вляют на работу.
