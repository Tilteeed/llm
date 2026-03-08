# LLM Lab Ansible
Ansible-проект для разворачивания и базовой настройки стенда под локальный или серверный запуск LLM-инфраструктуры на Ubuntu.

Проект покрывает:

- базовую подготовку Ubuntu-хоста;
- установку Docker;
- настройку NVIDIA Container Toolkit;
- развертывание Ollama в контейнере с поддержкой GPU;
- автоматическую загрузку моделей Ollama;
- развертывание monitoring stack:
  - Prometheus
  - Grafana
  - Node Exporter
- установку Garak;
- безопасную установку отдельного Python 3.11 runtime без изменения системного python3;
- создание отдельного virtual environment под LangChain-стек;
- создание административных пользователей платформы.

Проект подходит как для обычной Ubuntu VM с GPU, так и для WSL2 с GPU, если корректно выставлены переменные.

---

# Что умеет проект

На текущий момент проект включает следующие роли:

1. `ubuntu_base_nvidia_host`  
   Базовая подготовка Ubuntu-хоста, установка системных пакетов, Python tooling и, при необходимости, host NVIDIA driver.

2. `docker_engine`  
   Установка Docker Engine из официального репозитория Docker, настройка Docker service и групп пользователей.

3. `nvidia_container_toolkit`  
   Установка NVIDIA Container Toolkit и настройка Docker runtime для GPU-контейнеров.

4. `ollama_container`  
   Развертывание Ollama в Docker Compose с поддержкой GPU.

5. `ollama_pull_models`  
   Автоматическая загрузка моделей Ollama из списка переменных.

6. `monitoring_stack`  
   Развертывание monitoring stack на базе Prometheus, Grafana и Node Exporter.

7. `garak_runner`  
   Установка Garak в Python virtual environment.

8. `python_runtime`  
   Безопасная установка отдельного Python 3.11 runtime без изменения системного `python3`.

9. `langchain_runtime`  
   Создание отдельного Python virtual environment и установка LangChain-стека:
   - `langchain`
   - `langchain-community`
   - `langgraph`

10. `platform_users`  
   Создание пользователей платформы, выдача им sudo/docker-доступа и создание общей директории команды.

---

# Поддерживаемые сценарии

## 1. Обычная Ubuntu VM с NVIDIA GPU
Например, сервер или cloud VM с Tesla T4 / L4 / RTX и т.п.

В этом режиме проект:
- может установить host NVIDIA driver;
- выполняет стандартные Linux-проверки GPU;
- настраивает GPU внутри Docker-контейнеров.

## 2. WSL2 с GPU
В этом режиме:
- Linux host NVIDIA driver внутри Ubuntu **не устанавливается**;
- используется WSL-provided `nvidia-smi`;
- часть "железных" проверок отключается;
- GPU inside container продолжает работать через WSL2 passthrough.

## 3. Готовая cloud GPU VM

Этот сценарий подходит для облачной виртуальной машины, где уже предустановлены:

- NVIDIA driver
- Docker
- NVIDIA Container Toolkit
- другие базовые GPU-компоненты

В таком случае не всегда нужно запускать все роли подряд.

### Что обычно не требуется запускать повторно

Если образ уже подготовлен, можно не запускать:

- `ubuntu_base_nvidia_host`
- `docker_engine`
- `nvidia_container_toolkit`

### Что запускать поверх готового cloud image

Обычно достаточно запускать:

```bash
ansible-playbook site.yml --tags python_runtime -K
ansible-playbook site.yml --tags langchain_runtime -K
ansible-playbook site.yml --tags ollama_container -K
ansible-playbook site.yml --tags ollama_pull_models -K
ansible-playbook site.yml --tags monitoring_stack -K
ansible-playbook site.yml --tags garak_runner -K
ansible-playbook site.yml --tags platform_users -K
```

В group_vars/llm_nodes.yml обычно нужно указать:

```
ubuntu_nvidia_manage_host_driver: false
ubuntu_nvidia_smi_command: nvidia-smi
```
---

# Требования

## Хост
- Ubuntu 24.04 LTS или совместимая Ubuntu-среда
- Ansible target должен быть доступен по SSH или локально через `ansible_connection=local`
- Для GPU-сценария:
  - обычная Ubuntu VM с NVIDIA GPU  
    **или**
  - WSL2 с корректно работающим GPU passthrough

## Control machine
На машине, откуда запускается Ansible, должны быть:
- `ansible`
- SSH-доступ к target host
- права на `sudo` на target host

# Структура проекта

```
.
├── ansible.cfg
├── group_vars
│   └── llm_nodes.yml
├── inventory
│   └── hosts.ini
├── roles
│   ├── docker_engine
│   ├── garak_runner
│   ├── langchain_runtime
│   ├── monitoring_stack
│   ├── nvidia_container_toolkit
│   ├── ollama_container
│   ├── ollama_pull_models
│   ├── platform_users
│   ├── python_runtime
│   └── ubuntu_base_nvidia_host
└── site.yml
```

---

# Быстрый старт

## 1. Клонировать репозиторий

```bash
git clone git@github.com:Tilteeed/llm.git
cd llm
```

## 2. Проверить inventory

Открой:

```text
inventory/hosts.ini
```

### Пример для обычной VM по SSH

```ini
[llm_nodes]
llm-node-01 ansible_host=YOUR_HOST_IP ansible_user=ubuntu
```

### Пример для локального WSL2-теста

```ini
[llm_nodes]
llm-node-01 ansible_connection=local
```

---

## 3. Проверить `group_vars/llm_nodes.yml`

Это главный файл переменных проекта.

Именно в нем настраиваются:

* поведение под WSL2 или обычную VM;
* список моделей Ollama;
* параметры monitoring stack;
* пользователи платформы;
* и т.д.

---

## 4. Проверить подключение

### Для SSH-сценария

```bash
ansible llm_nodes -m ping -i inventory/hosts.ini -k -K
```

### Для локального WSL2-сценария

```bash
ansible llm_nodes -m ping -i inventory/hosts.ini -K
```

---

# Полный запуск проекта

Полный запуск всех ролей:

```bash
ansible-playbook site.yml -K
```

Где:

* `-K` — запросить sudo password на target host

---

# Рекомендуемый порядок запуска по ролям

Для первого запуска лучше идти поэтапно.

## 1. Базовая подготовка Ubuntu и GPU

```bash
ansible-playbook site.yml --tags ubuntu_base_nvidia_host -K
```

## 2. Docker

```bash
ansible-playbook site.yml --tags docker_engine -K
```

## 3. NVIDIA Container Toolkit

```bash
ansible-playbook site.yml --tags nvidia_container_toolkit -K
```

## 4. Ollama container

```bash
ansible-playbook site.yml --tags ollama_container -K
```

## 5. Загрузка моделей Ollama

```bash
ansible-playbook site.yml --tags ollama_pull_models -K
```

## 6. Monitoring stack

```bash
ansible-playbook site.yml --tags monitoring_stack -K
```

## 7. Garak

```bash
ansible-playbook site.yml --tags garak_runner -K
```

## 8. Python 3.11 runtime

```bash
ansible-playbook site.yml --tags python_runtime -K
```

## 9. LangChain runtime

```bash
ansible-playbook site.yml --tags langchain_runtime -K
```

## 10. Пользователи платформы
```bash
ansible-playbook site.yml --tags platform_users -K
```
---

# Запуск отдельных ролей

Ниже список основных тегов, которые используются в проекте.

## Основные role-level теги

* `ubuntu_base_nvidia_host`
* `docker_engine`
* `nvidia_container_toolkit`
* `ollama_container`
* `ollama_pull_models`
* `monitoring_stack`
* `garak_runner`
* `python_runtime`
* `langchain_runtime`
* `platform_users`

Пример:

```bash
ansible-playbook site.yml --tags ollama_container -K
```

---

# Настройка под обычную Ubuntu VM с GPU

Для обычной VM в `group_vars/llm_nodes.yml` должны быть значения такого типа:

```yaml
ubuntu_nvidia_manage_host_driver: true
ubuntu_nvidia_smi_command: nvidia-smi
```

Это означает:

* проект будет устанавливать host NVIDIA driver;
* роль будет использовать обычный `nvidia-smi`;
* после установки драйвера возможен reboot.

---

# Настройка под WSL2 с GPU

Для WSL2 в `group_vars/llm_nodes.yml` должны быть значения такого типа:

```yaml
ubuntu_nvidia_manage_host_driver: false
ubuntu_nvidia_smi_command: /usr/lib/wsl/lib/nvidia-smi
```

Это означает:

* Linux NVIDIA driver внутри Ubuntu не устанавливается;
* для проверки GPU используется WSL-provided `nvidia-smi`;
* часть low-level GPU-проверок пропускается.

---

# Проверка GPU

## На обычной VM

```bash
nvidia-smi
```

## На WSL2

```bash
/usr/lib/wsl/lib/nvidia-smi
```

Если таблица выводится корректно — GPU доступен.

---

# Проверка Docker

```bash
docker version
docker info
docker ps
```

Если `docker ps` не работает без `sudo`, значит текущий пользователь не состоит в группе `docker` или еще не перелогинился после добавления в группу.

---

# Проверка GPU inside Docker

```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Если команда показывает GPU внутри контейнера — `nvidia_container_toolkit` работает корректно.

---

# Проверка Ollama

## Проверка контейнера

```bash
docker ps
```

Убедись, что контейнер `ollama` запущен.

## Проверка API

```bash
curl http://127.0.0.1:11434/api/tags
```

## Проверка списка моделей

```bash
docker exec ollama ollama list
```

## Проверка генерации

```bash
docker exec -it ollama ollama run qwen2.5:7b
```

---

# Как понять, что Ollama реально использует GPU

Во время генерации модели открой отдельное окно и смотри:

## На обычной VM

```bash
watch -n 0.5 nvidia-smi
```

## На WSL2

```bash
watch -n 0.5 /usr/lib/wsl/lib/nvidia-smi
```

Обрати внимание на:

* `GPU-Util`
* занятую VRAM

Даже если `GPU-Util` не всегда высокий, занятая память GPU уже является важным индикатором того, что модель размещена в GPU.

---

# Настройка и проверка monitoring stack

Сейчас monitoring stack включает:

* Prometheus
* Grafana
* Node Exporter

`cAdvisor` можно включать или выключать через переменные проекта.

## Проверка Prometheus

```bash
curl http://127.0.0.1:9090/-/ready
```

Ожидаемый ответ:

```text
Prometheus Server is Ready.
```

## Проверка Grafana

```bash
curl http://127.0.0.1:3000/api/health
```

Ожидаемый ответ содержит:

```json
{
  "database": "ok"
}
```

## Проверка Node Exporter

```bash
curl http://127.0.0.1:9100/metrics | head
```

---

# Как открыть Grafana

Обычно:

* `http://127.0.0.1:3000`
* или `http://localhost:3000`

В WSL2 доступ по `localhost` может зависеть от сетевого режима WSL.
Если `localhost` не работает, используй IP WSL:

```bash
hostname -I
```

Потом открой в браузере:

```text
http://<IP_WSL>:3000
```

---

# Логин в Grafana

Логин и пароль задаются переменными:

```yaml
monitoring_stack_grafana_admin_user
monitoring_stack_grafana_admin_password
```

По умолчанию используется что-то вроде:

* login: `admin`
* password: `admin123`

После первого запуска пароль рекомендуется изменить и позже вынести в Ansible Vault.

---

# Garak

Роль `garak_runner` устанавливает Garak в отдельный virtual environment.

## Проверка установки

```bash
/opt/llm/garak/venv/bin/garak --help
```

## Пример запуска

```bash
/opt/llm/garak/venv/bin/garak
```

## Helper script

Если helper script включен, он создается по пути:

```text
/opt/llm/garak/bin/run_garak_ollama.sh
```
---

# Python 3.11 runtime

Проект включает отдельную роль `python_runtime` для безопасной установки Python 3.11 на Ubuntu.

## Зачем это нужно

На Ubuntu 20.04 системный Python часто слишком старый для современного LangChain-стека.  
При этом менять системный `python3` опасно, потому что можно сломать:

- `apt`
- системные Python-утилиты
- cloud image tooling

Поэтому проект использует отдельный Python runtime и не меняет `/usr/bin/python3`.

## Что делает роль `python_runtime`

- при необходимости подключает репозиторий для более новой версии Python;
- устанавливает Python 3.11;
- при необходимости может создать отдельный `venv`;
- не делает Python 3.11 системным Python по умолчанию.

## Запуск роли

```bash
ansible-playbook site.yml --tags python_runtime -K
```
## Проверка
```bash
python3.11 --version
```
### Ожидаемый результат — установленный Python 3.11.
---
# LangChain runtime

Проект включает отдельную роль langchain_runtime для создания isolated Python virtual environment и установки LangChain-стека.

## Что делает роль langchain_runtime

создает отдельный venv;

обновляет внутри него:

- pip

- setuptools

- wheel

устанавливает:

- langchain

- langchain-community

- langgraph

## Почему это сделано отдельной ролью

Это позволяет:

 - не смешивать прикладные Python-пакеты с системным Python;

 - безопасно обновлять библиотечный стек;

 - держать LangChain-окружение отдельно от Garak и других системных компонентов;

 - легко повторять установку на других хостах.

## Запуск роли
```bash
ansible-playbook site.yml --tags langchain_runtime -K
```
## Проверка
```bash
/opt/venvs/langchain311/bin/python -c "import langchain; import langchain_community; import langgraph; print('ok')"
```

Если выводится ok, значит LangChain runtime установлен корректно.

## Активация venv вручную
```bash
source /opt/venvs/langchain311/bin/activate
python --version
pip list
```
## Выход из venv
```bash
deactivate
```

## Установка дополнительных Python-пакетов

Дополнительные пакеты можно указать в group_vars/llm_nodes.yml.

Пример:
langchain_runtime_extra_packages:
  - langchain-openai
  - chromadb
  - faiss-cpu

## После изменения переменных роль запускается повторно:

```bash
ansible-playbook site.yml --tags langchain_runtime -K
```


# Пользователи платформы

Роль `platform_users` создает пользователей, задает им пароли и добавляет в нужные группы.

## Что настраивает роль

* системные пользователи
* домашние каталоги
* membership в `sudo`
* membership в `docker`
* membership в общей группе команды
* общую директорию `/opt/llm/shared`

## Проверка пользователей

```bash
getent passwd Alex IvanO IvanM ilya Dima
id Alex
id IvanO
id IvanM
id ilya
id Dima
ls -ld /opt/llm/shared
```

---

# Переменные проекта

Все основные переменные лежат в:

```text
group_vars/llm_nodes.yml
```

Они разбиты по блокам ролей:

* `ubuntu_base_nvidia_host`
* `docker_engine`
* `nvidia_container_toolkit`
* `ollama_container`
* `ollama_pull_models`
* `monitoring_stack`
* `garak_runner`
* `python_runtime`
* `langchain_runtime`
* `platform_users`

Если требуется изменить поведение роли, в первую очередь смотри именно туда.

---

# Безопасность

## 1. Пароли пользователей

Сейчас пользователи платформы используют `password_hash`.
Это лучше, чем хранить пароль в открытом виде, но в будущем рекомендуется вынести такие значения в **Ansible Vault**.

## 2. Пароль Grafana

Тоже лучше позже вынести в Vault.

## 3. SSH keys

Сейчас возможен вход по паролю, но в production или более стабильной среде лучше перейти на SSH keys.

## 4. Docker group

Пользователь в группе `docker` фактически получает очень высокий уровень доступа. Используй это осознанно.

---

# Ограничения и текущие особенности

## 1. WSL2

Под WSL2 есть ряд особенностей:

* host NVIDIA driver внутри Ubuntu не ставится;
* `nvidia-smi` обычно вызывается через `/usr/lib/wsl/lib/nvidia-smi`;
* `localhost`-доступ к сервисам из Windows может зависеть от конфигурации сети WSL.

## 2. Monitoring dashboards

Auto-provisioning дашбордов Grafana требует валидных JSON-файлов.
Если JSON пустой или кривой, Grafana покажет ошибки provisioning.

## 3. cAdvisor

`cAdvisor` можно временно отключать, если он не нужен или если нужен более простой monitoring stack.

---

# Типичные проблемы

## Docker не работает без sudo

Проверь, что пользователь находится в группе `docker`:

```bash
id <username>
```

Если пользователь только что добавлен в группу, нужно перелогиниться.

---

## `nvidia-smi` не находится в WSL2

Используй:

```bash
/usr/lib/wsl/lib/nvidia-smi
```

---

## Prometheus target `ollama` в статусе DOWN

Если в Prometheus добавлен scrape target для Ollama, а сам Ollama не отдает `/metrics`, target будет красным.

Это не значит, что Ollama упал.
Это значит, что у сервиса нет Prometheus-compatible endpoint по `/metrics`.

---

## Grafana видит datasource, но дашборды пустые

Обычно причина одна из:

* dashboard ожидает другие labels/metrics
* в JSON зашит placeholder datasource
* datasource был неправильно выбран или не сохранен
* Prometheus реально не скрапит нужные метрики

---
## Python 3.11 не находится

Проверь:

```bash
python3.11 --version
```
Если команда не находится, значит роль python_runtime еще не запускалась или установка завершилась с ошибкой.

## LangChain не импортируется

Проверь:
```bash
/opt/venvs/langchain311/bin/python -c "import langchain; import langchain_community; import langgraph; print('ok')"
```
Если импорт падает, нужно повторно выполнить:
```bash
ansible-playbook site.yml --tags langchain_runtime -K
```
### Не нужно менять системный python3
Проект специально не меняет /usr/bin/python3.
Это сделано для безопасности и совместимости с Ubuntu, apt и cloud image tooling.
---
# Развитие проекта

Что можно улучшить дальше:

* auto-provision datasource и dashboard JSON для Grafana
* доработка monitoring под GPU-метрики
* Open WebUI
* отдельные inventory/group_vars под:

  * WSL2
  * cloud GPU VM
  * bare metal
* вынос секретов в Ansible Vault
* переход пользователей на SSH keys
* более аккуратные production-safe defaults

---

# Пример полного запуска по шагам

```bash
ansible-playbook site.yml --tags ubuntu_base_nvidia_host -K
ansible-playbook site.yml --tags docker_engine -K
ansible-playbook site.yml --tags nvidia_container_toolkit -K
ansible-playbook site.yml --tags ollama_container -K
ansible-playbook site.yml --tags ollama_pull_models -K
ansible-playbook site.yml --tags monitoring_stack -K
ansible-playbook site.yml --tags garak_runner -K
ansible-playbook site.yml --tags python_runtime -K
ansible-playbook site.yml --tags langchain_runtime -K
ansible-playbook site.yml --tags platform_users -K
```

---

# Автор / назначение

Проект предназначен для быстрого развертывания лабораторного LLM-стенда с упором на:

* простоту структуры;
* понятные роли;
* подробные комментарии в Ansible;
* возможность тестировать как на обычной Ubuntu VM с GPU, так и на WSL2.

---

