# DevOps Journey: Путь от разработки до продакшена

Этот репозиторий демонстрирует полный DevOps workflow - от создания приложения до развертывания в Kubernetes. Практический опыт работы с основными инструментами DevOps: Git, веб-серверы, контейнеризация, управление конфигурацией и оркестрация. Чтобы начать наш путь, потребуется установить Ubuntu 22.04, так как все примеры основаны на ней.

## Содержание
1. [Начальная настройка и разработка](#начальная-настройка-и-разработка)
2. [Система контроля версий Git](#система-контроля-версий-git)
3. [Настройка веб-серверов](#настройка-веб-серверов)
4. [Инфраструктура как код](#инфраструктура-как-код)
5. [Управление конфигурацией с Ansible](#управление-конфигурацией-с-ansible)
6. [Контейнеризация с Docker](#контейнеризация-с-docker)
7. [Оркестрация контейнеров с Kubernetes](#оркестрация-контейнеров-с-kubernetes)

---

## Начальная настройка и разработка

### Подготовка окружения
Настройка среды разработки Ubuntu и создание структуры проекта:

```bash
# Создание каталога проекта
mkdir hello
cd hello
ls
```

**Зачем это нужно**: Правильная организация проекта критически важна для поддержки и командной работы в DevOps практиках. Структурированный подход с самого начала экономит время в будущем.

### Python веб-приложение
Создание простого Python приложения для демонстрации pipeline развертывания:

```python
#!/usr/bin/env python3
import datetime

def do_magic():
    now = datetime.datetime.now()
    return "Hello! {0}".format(now)

if __name__ == "__main__":
    print(do_magic())
```

```bash
chmod +x ./app.py
./app.py
```

**Назначение**: Простое приложение служит нашим артефактом развертывания. Несмотря на простоту, оно демонстрирует реальные концепции развертывания: генерация динамического контента и правильные права доступа к файлам.

---

## Система контроля версий Git

### Инициализация репозитория
Внедрение системы контроля версий с самого начала проекта:

```bash
# Установка и инициализация Git
sudo apt install git
git init
```

**Первоначальная проблема**: Git не распознает каталог как репозиторий:
```bash
git status
# fatal: not a git repository (or any of the parent directories): .git
```

**Зачем нужен Git**: Git - это система управления исходным кодом, которая позволяет отслеживать изменения, работать в команде и легко возвращаться к предыдущим версиям кода.

### Настройка пользователя Git
Обязательная конфигурация для идентификации автора изменений:

```bash
git config --global user.email "alm15@mail.ru"
git config --global user.name "Almaskhan Oskenbayev"
```

**Важность**: Каждый коммит должен содержать информацию о том, кто его сделал. Без этих настроек Git не позволит сохранить изменения.

### Первый коммит
Добавление файлов в систему контроля версий:

```bash
git add app.py
git commit -m 'first version'
```

**Принцип работы**: 
- `git add` - добавляет файлы в область подготовки (staging area)
- `git commit` - создает снимок текущего состояния с описанием

### Просмотр истории изменений
```bash
# Краткий просмотр коммитов
git log

# Детальный просмотр с изменениями
git log -p
```

**Практическое применение**: История коммитов показывает эволюцию кода, кто и когда внес изменения. Это критически важно для отладки и понимания развития проекта.

---

## Настройка веб-серверов

### Apache2 с CGI

#### Установка и базовая настройка
```bash
sudo apt install apache2
```

**Назначение**: Apache2 - надежный веб-сервер для обработки HTTP-запросов. CGI (Common Gateway Interface) позволяет выполнять скрипты на сервере.

#### Настройка прав доступа
```bash
# Добавление www-data в группу пользователя для доступа к файлам
sudo usermod www-data -a -G user

# Создание символической ссылки на каталог проекта
cd /var/www/
sudo mv html html2
sudo ln -s ~/hello/ html
```

**Проблема и решение**: По умолчанию Apache не может читать файлы из домашнего каталога пользователя. Добавление www-data в группу пользователя решает эту проблему. Чтобы узнать группу в которой состоит текущий пользователь введите команды whoami или id. И далее замените user на полученное значение в команде usermod.

#### Включение CGI
```bash
sudo a2enmod cgi
sudo vim /etc/apache2/sites-enabled/000-default.conf
```

Добавить в конфигурацию:
```apache
<Directory /var/www/html>
    AllowOverride All
<Directory>
```

#### Настройка .htaccess
```bash
# Создание файла .htaccess для обработки Python как CGI
cat > .htaccess << EOF
AddHandler cgi-script .py
Options +ExecCGI
DirectoryIndex app.py
EOF
```

#### Модификация приложения для CGI
```python
#!/usr/bin/env python3
import datetime
import os

def do_magic():
    now = datetime.datetime.now()
    return "Hello! {0}".format(now)

if __name__ == "__main__":
    # Определяем, вызывается ли скрипт веб-сервером
    if 'REQUEST_URI' in os.environ:
        print("Content-type: text/html\n\n")
    print(do_magic())
```

**Объяснение логики**: Проверка переменной окружения `REQUEST_URI` позволяет сценарию работать как в командной строке, так и через веб-сервер.

### Переход на uWSGI

**Почему uWSGI лучше CGI**: CGI создает новый процесс для каждого запроса, что неэффективно. uWSGI (WSGI сервер) использует пулы процессов и потоков, обеспечивая лучшую производительность.

#### Создание ветки для разработки
```bash
git checkout -b uwsgi
```

**Зачем нужны ветки**: Ветки позволяют разрабатывать новые функции изолированно, не мешая основной работе команды.

#### Модификация для WSGI
```python
def application(env, start_response):
    start_response('200 OK',[('Content-Type','text/html')])
    return [do_magic().encode()]
```

#### Установка и настройка uWSGI
```bash
sudo apt install uwsgi-plugin-python3

# Создание конфигурационного файла dev.ini
cat > dev.ini << EOF
[uwsgi]
plugin=python3
http-socket=:9090
wsgi-file=app.py
EOF

# Запуск сервера разработки
uwsgi dev.ini
```

#### Слияние веток
```bash
git checkout master
git merge uwsgi
```

**Принцип GitFlow**: После завершения разработки функции, ветка сливается в основную ветку.

### Nginx + uWSGI в продакшене

**Зачем нужен Nginx**: Nginx эффективно обрабатывает статические файлы и проксирует динамические запросы к uWSGI.

#### Установка и настройка Nginx
```bash
# Отключение Apache2
sudo systemctl disable apache2
sudo systemctl stop apache2

# Установка Nginx
sudo apt install nginx -y

# Создание конфигурации
sudo vi /etc/nginx/sites-available/hello.conf
```

Конфигурация hello.conf:
```nginx
server {
    listen 80;
    root /var/www/html;
    location / {
        include /etc/nginx/uwsgi_params;
        uwsgi_pass 127.0.0.1:9000;
        uwsgi_param Host $host;
        uwsgi_param X-Real-IP $remote_addr;
        uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
        uwsgi_param X-Forwarded-Proto $http_x_forwarded_proto;
    }
}
```

#### Активация конфигурации
```bash
# Перемещение в правильное место
sudo mv /etc/nginx/sites-enabled/hello.conf /etc/nginx/sites-available/
sudo ln -s ../sites-available/hello.conf /etc/nginx/sites-enabled/

# Удаление конфигурации по умолчанию
sudo rm /etc/nginx/sites-enabled/default
```

#### Продакшн конфигурация uWSGI
```bash
# Создание prod.ini
cat > prod.ini << EOF
[uwsgi]
plugin=python3
socket=127.0.0.1:9000
wsgi-file=app.py
EOF
```

**Разница dev/prod**: В разработке используется HTTP-сокет для прямого доступа, в продакшене - Unix/TCP сокет для связи с Nginx.

---

## Инфраструктура как код

### Организация файлов развертывания
```bash
# Создание структуры для инфраструктурного кода
sudo mkdir -p deploy/{apache2,uwsgi,nginx,systemd}

# Перемещение конфигурационных файлов
sudo mv .htaccess deploy/apache2/
sudo mv *.ini deploy/uwsgi/
sudo cp /etc/nginx/sites-available/hello.conf deploy/nginx/
```

**Принцип Infrastructure as Code**: Все настройки инфраструктуры хранятся в репозитории вместе с кодом приложения. Это обеспечивает воспроизводимость развертываний.

### Создание systemd service
```bash
sudo vi /etc/systemd/system/hello.service
```

```ini
[Unit]
Description=Hello app
Requires=network.target
After=network.target

[Service]
TimeoutStartSec=0
RestartSec=10
Restart=always
WorkingDirectory=/opt/hello
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all
ExecStart=/usr/bin/uwsgi deploy/uwsgi/prod.ini

[Install]
WantedBy=multi-user.target
```

**Зачем systemd**: Systemd управляет сервисами системы, обеспечивает автозапуск, мониторинг и автоматический перезапуск при сбоях.

### Настройка GitHub репозитория
```bash
# Настройка SSH ключей для безопасного доступа
ssh-keygen -t rsa -b 4096 -C "alm15@mail.ru"
ssh-add ~/.ssh/id_rsa

# Добавление удаленного репозитория
git remote add origin git@github.com:almsys/hello.git
git push origin master

# Создание тега версии
git tag v1.0
git push origin v1.0
```

**Теги версий**: Позволяют легко идентифицировать и развертывать конкретные версии приложения.

---

## Управление конфигурацией с Ansible

### Установка LXD для тестирования
```bash
sudo lxd init
# Выбрать 'dir' для storage backend, остальное по умолчанию
```

**Зачем контейнеры**: Контейнеры обеспечивают изолированную среду для тестирования развертывания без влияния на основную систему.

### Настройка профиля LXD
```bash
lxc profile edit default
```

Добавить в конфигурацию:
```yaml
config:
  cloud-init.user-data: |
    #cloud-config
    packages:
      - openssh-server
      - vim
    ssh_pwauth: false
    ssh_authorized_keys:
      - "ssh-rsa YOUR_PUBLIC_KEY"
```

### Установка Ansible
```bash
sudo apt update -y
sudo apt install -y software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible -y
```

**Роль Ansible в DevOps**: Ansible автоматизирует настройку серверов, обеспечивая согласованность окружений и сокращая время развертывания.

### Создание инвентаря Ansible
```bash
mkdir ansible
cd ansible

# ansible.cfg
cat > ansible.cfg << EOF
[defaults]
inventory = hosts
host_key_checking = False
interpreter_python=auto_silent
EOF

# hosts
cat > hosts << EOF
test ansible_host=10.169.189.127 ansible_user=ubuntu
EOF
```

### Ansible Playbook для развертывания
```yaml
---
- name: deploy hello app
  gather_facts: false
  hosts: test
  become: true
  vars:
    repo: https://github.com/almsys/hello
    repo_dir: '/opt/hello'
    packages:
      - nginx
      - python3
      - uwsgi-plugin-python3
  
  tasks:
    - name: update apt cache
      apt:
        update_cache: yes
    
    - name: install packages
      package:
        name: "{{ item }}"
        state: latest
      with_items: "{{ packages }}"
    
    - name: checkout repo
      git:
        repo: "{{ repo }}"
        dest: "{{ repo_dir }}"
        version: v2.0
    
    - name: copy systemd config
      copy:
        remote_src: yes
        src: "{{ repo_dir }}/deploy/systemd/hello.service"
        dest: "/etc/systemd/system/hello.service"
    
    - name: enable and start service
      systemd:
        name: hello
        daemon_reload: yes
        enabled: yes
        state: started
    
    - name: copy nginx config
      copy:
        remote_src: yes
        src: "{{ repo_dir }}/deploy/nginx/hello.conf"
        dest: "/etc/nginx/sites-available/hello.conf"
    
    - name: disable default nginx config
      file:
        state: absent
        path: /etc/nginx/sites-enabled/default
      notify: restart nginx
    
    - name: enable our nginx config
      file:
        src: /etc/nginx/sites-available/hello.conf
        dest: /etc/nginx/sites-enabled/hello.conf
        state: link
      notify: restart nginx
  
  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
```

### Запуск развертывания
```bash
# Проверка синтаксиса
ansible-playbook --syntax-check deploy.yml

# Выполнение развертывания
ansible-playbook deploy.yml

# Проверка результата
curl http://10.169.189.127
```

**Преимущества Ansible**: Декларативный подход - описываем желаемое состояние, Ansible делает все необходимые изменения для его достижения.

---

## Контейнеризация с Docker

### Установка Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker user
# Необходим logout/login для применения изменений
```

### Создание Dockerfile
```dockerfile
FROM tiangolo/uwsgi-nginx:python3.7
COPY ./app.py /app/app.py
COPY ./deploy/uwsgi/prod.ini /app/uwsgi.ini
WORKDIR /app
```

**Преимущества готовых образов**: Используя образ tiangolo/uwsgi-nginx, мы получаем предварительно настроенную среду, что экономит время и обеспечивает стабильность.

### Сборка и запуск контейнера
```bash
# Сборка образа
docker build -t almsys/hello:1.0 .

# Запуск контейнера
docker container run --publish 80:80 --detach almsys/hello:1.0
```

### Публикация в Docker Hub
```bash
docker login --username almsys
docker push almsys/hello:1.0
```

**Глобальная доступность**: После публикации в Docker Hub контейнер можно запустить на любой системе с Docker.

### Docker Compose
```yaml
version: "3"
services:
  hello:
    image: almsys/hello:1.0
    ports:
      - "80:80"
    restart: always
```

```bash
docker-compose up
```

**Упрощение управления**: Docker Compose позволяет описать всю архитектуру приложения в одном файле и управлять ей простыми командами.

---

## Оркестрация контейнеров с Kubernetes

### Установка Minikube и kubectl
```bash
# Установка Minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
sudo mkdir -p /usr/local/bin/
sudo install minikube /usr/local/bin/

# Установка kubectl
sudo snap install kubectl --classic

# Запуск кластера
minikube start
```

### Kubernetes манифест
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
  - protocol: "TCP"
    port: 80
    targetPort: 80
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
spec:
  selector:
    matchLabels:
      app: hello
  replicas: 4
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: maniaque/hello:1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 80
```

### Развертывание в Kubernetes
```bash
# Применение конфигурации
kubectl apply -f k8s.yml

# Проверка подов
kubectl get pods

# Проверка сервисов
kubectl get svc

# Доступ к приложению
kubectl cluster-info
curl http://192.168.49.2:31820
```

**Высокая доступность**: Kubernetes автоматически управляет 4 репликами приложения, обеспечивая отказоустойчивость и балансировку нагрузки.

---

## Заключение

Этот проект демонстрирует полный DevOps pipeline:

1. **Разработка** - создание простого, но функционального приложения
2. **Версионирование** - использование Git для отслеживания изменений
3. **Веб-серверы** - эволюция от CGI к современным решениям
4. **Автоматизация** - Ansible для управления конфигурацией
5. **Контейнеризация** - Docker для упаковки приложения
6. **Оркестрация** - Kubernetes для масштабирования
