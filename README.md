# DevOps Monitoring Stack

Автоматизированная настройка стека мониторинга с использованием Vagrant, Ansible, Prometheus, Grafana и Node Exporter.

## Компоненты

- **Vagrant**: создание виртуальной машины Ubuntu 20.04
- **Ansible**: автоматизация настройки компонентов
- **Node Exporter**: сбор метрик системы
- **Prometheus**: хранение и обработка метрик
- **Grafana**: визуализация метрик с дашбордом Node Exporter Full

## Запуск

```bash
git clone https://github.com/ipchel/devops-test.git
cd devops-test
vagrant up
```

После завершения настройки:
- Grafana: http://localhost:3000 (логин: admin, пароль: admin)
- Prometheus: http://localhost:9090

## Детали реализации

- Node Exporter работает как системный сервис с автозапуском
- Prometheus и Grafana запускаются в контейнерах Docker
- Мониторинг настроен на автоматическое восстановление после перезагрузки
- Prometheus настроен для сбора метрик с Node Exporter
- Grafana содержит готовый дашборд Node Exporter Full
