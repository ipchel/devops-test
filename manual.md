# Техническая документация: Внедрение системы мониторинга на базе Prometheus и Grafana

## 1. Введение

Данная документация описывает процесс внедрения автоматизированной системы мониторинга инфраструктуры с использованием инструментов с открытым исходным кодом: Node Exporter, Prometheus и Grafana. Система внедряется посредством автоматизации на базе Vagrant и Ansible на виртуальной машине с Ubuntu 20.04.

## 2. Архитектура решения

Решение состоит из следующих компонентов:
- **Виртуальная машина**: Ubuntu 20.04 LTS, развертываемая через Vagrant
- **Node Exporter**: системный сервис, собирающий метрики о состоянии операционной системы
- **Prometheus**: сервер для сбора и хранения временных рядов метрик, запущенный в Docker-контейнере
- **Grafana**: платформа визуализации метрик, запущенная в Docker-контейнере

Взаимодействие компонентов:
1. Node Exporter собирает системные метрики хоста и экспортирует их на порту 9100
2. Prometheus собирает метрики с Node Exporter с заданным интервалом и сохраняет их в базе данных временных рядов
3. Grafana подключается к Prometheus как к источнику данных и визуализирует полученные метрики через предустановленные дашборды

## 3. Предварительные требования

Для развертывания решения необходимы:
- Vagrant 2.2.x или выше
- VirtualBox 6.x или выше
- Git
- Доступ к интернету для загрузки образов

## 4. Настройка инфраструктуры

### 4.1. Конфигурация Vagrant

Создан Vagrantfile для определения виртуальной машины:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.hostname = "ubuntu2004.localdomain"
  
  # Проброс портов
  config.vm.network "forwarded_port", guest: 9090, host: 9090
  config.vm.network "forwarded_port", guest: 3000, host: 3000
  
  # Shared folder
  config.vm.synced_folder ".", "/vagrant", disabled: false
  
  # Provisioning
  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "playbook.yml"
    ansible.become = true
  end
end
```

### 4.2. Ansible-плейбук для автоматизации настройки

Разработан Ansible-плейбук для автоматизации процесса настройки всех компонентов мониторинга. Он выполняет следующие задачи:
- Установка необходимых пакетов
- Настройка Docker и docker-compose
- Установка и настройка Node Exporter как системного сервиса
- Конфигурация Prometheus для сбора метрик
- Настройка Grafana и установка дашбордов

## 5. Внедрение Node Exporter

### 5.1. Установка Node Exporter

```yaml
- name: Download Node Exporter
  get_url:
    url: https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
    dest: /tmp/node_exporter.tar.gz

- name: Extract Node Exporter
  unarchive:
    src: /tmp/node_exporter.tar.gz
    dest: /tmp
    remote_src: yes

- name: Move Node Exporter binary
  copy:
    src: /tmp/node_exporter-1.5.0.linux-amd64/node_exporter
    dest: /usr/local/bin/
    mode: 0755
    remote_src: yes
```

### 5.2. Настройка службы systemd

```yaml
- name: Create Node Exporter systemd service
  copy:
    content: |
      [Unit]
      Description=Node Exporter
      After=network.target
      
      [Service]
      User=root
      ExecStart=/usr/local/bin/node_exporter
      Restart=always
      
      [Install]
      WantedBy=multi-user.target
    dest: /etc/systemd/system/node_exporter.service

- name: Start and enable Node Exporter
  systemd:
    name: node_exporter
    state: started
    enabled: yes
    daemon_reload: yes
```

## 6. Развертывание Prometheus

### 6.1. Конфигурация Prometheus

Файл `prometheus.yml` для определения параметров сбора метрик:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node'
    static_configs:
      - targets: ['172.17.0.1:9100']
```

При настройке Prometheus в контейнере Docker необходимо указывать IP-адрес хост-машины (172.17.0.1), доступный из контейнера, а не localhost.

### 6.2. Docker Compose для Prometheus

В файле docker-compose.yml определена конфигурация Prometheus:

```yaml
prometheus:
  image: prom/prometheus
  ports:
    - "9090:9090"
  volumes:
    - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
  restart: always
```

## 7. Настройка Grafana

### 7.1. Конфигурация источника данных

Создана конфигурация источника данных Prometheus для Grafana в формате YAML:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### 7.2. Установка дашборда Node Exporter Full

Для Grafana импортирован готовый дашборд Node Exporter Full, доступный в официальном репозитории Grafana.

```yaml
- name: Download Node Exporter Full dashboard
  get_url:
    url: https://grafana.com/api/dashboards/1860/revisions/27/download
    dest: ~/monitoring/grafana/provisioning/dashboards/node_exporter_full.json
```

### 7.3. Docker Compose для Grafana

```yaml
grafana:
  image: grafana/grafana
  ports:
    - "3000:3000"
  volumes:
    - ./grafana/provisioning:/etc/grafana/provisioning
    - grafana-storage:/var/lib/grafana
  environment:
    - GF_SECURITY_ADMIN_PASSWORD=admin
    - GF_USERS_ALLOW_SIGN_UP=false
  restart: always
```

## 8. Тестирование и верификация

### 8.1. Проверка доступности Node Exporter

```bash
systemctl status node_exporter
curl localhost:9100/metrics | head
```

### 8.2. Проверка работы Prometheus

```bash
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/targets
```

### 8.3. Проверка доступности Grafana

```bash
curl -I http://localhost:3000
```

## 9. Проверка отказоустойчивости

Для проверки корректного автоматического запуска системы мониторинга после перезагрузки виртуальной машины выполнен reboot с последующей верификацией всех компонентов:

```bash
# После перезагрузки
systemctl status node_exporter
docker ps
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/api/v1/targets
curl -I http://localhost:3000
```

Подтверждено, что все компоненты возобновляют работу автоматически.

## 10. Управление кодом

Весь код размещен в Git-репозитории с использованием SSH-аутентификации:

```bash
git init
git add .
git commit -m "Initial commit: Monitoring stack setup"
git remote add origin git@github.com:ipchel/devops-test.git
git push -u origin main
```

## 11. Рекомендации по эксплуатации

### 11.1. Доступ к интерфейсам

- Grafana: http://localhost:3000 (логин: admin, пароль: admin)
- Prometheus: http://localhost:9090

### 11.2. Техническое обслуживание

- Регулярно проверять работоспособность всех компонентов
- Мониторить свободное место для хранения метрик Prometheus
- Рекомендуемый интервал обновления компонентов: 1 раз в 6 месяцев

## 12. Заключение

Данная система мониторинга предоставляет базовую инфраструктуру для наблюдения за состоянием виртуальной машины. Система полностью автоматизирована, отказоустойчива и разворачивается одной командой.
