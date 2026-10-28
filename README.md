# Проектная работа по теме "Реализация веб-проекта с использованием оркестрации Ansible и автоматическим восстановлением после сбоя".

## Цели проектной работы:

закрепить и продемонстрировать полученные знания и навыки;<br>

создать полноценный веб-проект;<br>

подготовить проект, который можно использовать как часть профессионального портфолио.<br>



Веб проект с развертыванием нескольких виртуальных машин должен отвечать следующим требованиям:<br>

включен https;<br>
основная инфраструктура в DMZ зоне;<br>
файрвалл на входе;<br>
сбор метрик и настроенный алертинг;<br>
организован централизованный сбор логов;<br>
организован backup.<br>
В моей реализации дополнительно реализованы отказоустойчивость web-части, MariaDB Primary/Replica, GlusterFS и автоматизированное восстановление после полного удаления отдельных VM.<br>




## Структура проекта

```text
.
├── Vagrantfile
├── ansible.cfg
├── inventory/
│   └── production/
├── playbooks/
│   ├── site.yml
│   ├── recover-backend.yml
│   ├── recover-database.yml
│   └── recover-monitoring.yml
├── roles/
│   ├── common/
│   ├── wordpress/
│   ├── wordpress_storage/
│   ├── nginx_frontend/
│   ├── mariadb/
│   ├── mariadb_replication/
│   ├── gluster_recovery/
│   ├── gluster_arbiter_recovery/
│   ├── gluster_metadata_backup/
│   ├── mariadb_recovery/
│   ├── mariadb_primary_recovery/
│   ├── monitoring_state_backup/
│   ├── monitoring_state_restore/
│   ├── node_exporter/
│   ├── mysqld_exporter/
│   ├── alloy/
│   ├── backup/
│   └── firewall_*/
└── backups/
```




## Выполнение

### Стенд: Nginx + WordPress (PHP-FPM) + Django (Gunicorn/WSGI) + Node.js
```
Разворачивала вручную на своём компьютере через Docker Desktop / docker-compose.
```


## Архитектура

Проект состоит из шести виртуальных машин:

| VM | Назначение |
|---|---|
| `frontend` | Nginx, HTTPS, reverse proxy, балансировка |
| `backend01` | Apache + PHP + WordPress, GlusterFS brick |
| `backend02` | Apache + PHP + WordPress, GlusterFS brick |
| `db01` | MariaDB PRIMARY |
| `db02` | MariaDB REPLICA |
| `monitoring` | Prometheus, Grafana, Loki, Alertmanager, Blackbox Exporter, GlusterFS arbiter |

Сети разделены по назначению:

- management network — управление VM;
- application network — frontend ↔ backend;
- database network — backend ↔ MariaDB.

Публичной точкой входа является `frontend`. Внутренние backend- и database-узлы доступны только по необходимым направлениям, ограниченным `nftables`.

## Соответствие требованиям OTUS

| Требование | Реализация |
|---|---|
| Несколько VM | 6 VM под управлением Vagrant |
| HTTPS | Nginx + TLS, HTTP → HTTPS |
| Сегментация инфраструктуры | отдельная точка входа, application и database сети |
| Firewall | `nftables` на всех узлах |
| Метрики | Prometheus, Node Exporter, mysqld_exporter, Blackbox Exporter |
| Alerting | Alertmanager + Telegram |
| Централизованные логи | Grafana Alloy → Loki |
| Визуализация | Grafana |
| Backup | MariaDB, WordPress, GlusterFS metadata, monitoring state |
| Автоматизация | Ansible |
| Восстановление | проверено после полного удаления VM |

## Используемые технологии

### Инфраструктура и автоматизация

- Vagrant
- VirtualBox
- Ansible
- Jinja2
- Ansible Vault
- systemd

### Web

- Nginx
- Apache HTTP Server
- PHP
- WordPress
- WP-CLI
- HTTPS

### Данные

- MariaDB Primary/Replica
- GTID replication
- binary log
- relay log
- GlusterFS Replica 2 + Arbiter 1

### Monitoring и logging

- Prometheus
- Node Exporter
- mysqld_exporter
- Blackbox Exporter
- Grafana
- Alertmanager
- Telegram
- Grafana Alloy
- Loki

## Развёртывание

Запуск VM:

```bash
vagrant up
```

Проверка Ansible:

```bash
ansible all -m ping
```

Полное конфигурирование:

```bash
ansible-playbook playbooks/site.yml
```

После успешного запуска в `PLAY RECAP` ожидается:

```text
failed=0
unreachable=0
```

## Доступ к сайту

Добавить в `/etc/hosts`:

```text
192.168.56.10 project.local
```

Сайт:

```text
https://project.local
```

## Web HA

Nginx принимает HTTPS и балансирует запросы:

```text
Пользователь
    |
    v
frontend
    |
    +--> backend01
    |
    +--> backend02
```

При потере одного backend сайт продолжает работать через второй.

## WordPress и GlusterFS

Оба backend используют общий `wp-content`.

Состав GlusterFS:

```text
backend01  -> data brick
backend02  -> data brick
monitoring -> arbiter brick
```

Проверка:

```bash
sudo gluster volume status wp-content
```

Ожидается:

```text
Online: Y
```

Проверка heal:

```bash
sudo gluster volume heal wp-content info
```

После синхронизации:

```text
Number of entries: 0
```

## MariaDB replication

Схема:

```text
db01 -> PRIMARY
db02 -> REPLICA
```

Проверка replica:

```bash
vagrant ssh db02
```

```bash
sudo mariadb -e "SHOW SLAVE STATUS\G"
```

Ожидается:

```text
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: 0
```

## Monitoring

На `monitoring` работают:

- Prometheus;
- Grafana;
- Loki;
- Alertmanager;
- Blackbox Exporter;
- Node Exporter;
- Alloy.

Контролируются:

- доступность VM;
- CPU;
- RAM;
- свободное место;
- доступность сайта;
- backend;
- MariaDB;
- replication;
- replication lag.

## Централизованный сбор логов

```text
VM logs
  |
  v
Grafana Alloy
  |
  v
Loki
  |
  v
Grafana
```

Собираются systemd journal, Nginx logs и Apache logs.

## Firewall

На всех узлах используется `nftables`.

Основная политика:

```text
input   -> ограничен
forward -> ограничен
output  -> разрешён
```

Разрешены только необходимые соединения между сервисами и сетевыми сегментами.

## Backup

Автоматические backup запускаются через systemd timers.

Резервируются:

- MariaDB;
- WordPress;
- GlusterFS metadata;
- Prometheus;
- Loki;
- Grafana.

Внешние копии хранятся через `/vagrant/backups` на хостовой системе.

## Восстановление после полного удаления VM

### Frontend

```bash
vagrant destroy -f frontend
vagrant up frontend
ansible frontend -m ping
ansible-playbook playbooks/site.yml --limit frontend
```

Отдельный recovery playbook не нужен, потому что `frontend` не содержит уникального persistent state.

### Backend

```bash
vagrant destroy -f backend01
vagrant up backend01
ansible backend01 -m ping
ansible-playbook playbooks/recover-backend.yml --limit backend01
```

Для `backend02` используется тот же playbook с `--limit backend02`.

Recovery восстанавливает Gluster metadata, UUID, brick, membership, heal, mount `wp-content`, Apache, monitoring и logging.

### db02

```bash
vagrant destroy -f db02
vagrant up db02
ansible db02 -m ping
ansible-playbook playbooks/recover-database.yml --limit db02
```

Данные replica восстанавливаются с работающего `db01`, затем автоматически настраивается GTID replication.

### db01

```bash
vagrant destroy -f db01
vagrant up db01
ansible db01 -m ping
ansible-playbook playbooks/recover-database.yml --limit db01
```

Данные primary восстанавливаются с surviving replica `db02`, после чего восстанавливается исходная схема replication.

### Monitoring

```bash
vagrant destroy -f monitoring
vagrant up monitoring
ansible monitoring -m ping
ansible-playbook playbooks/recover-monitoring.yml --limit monitoring
```

Восстанавливаются сервисы monitoring, Gluster arbiter и историческое состояние Prometheus/Loki/Grafana.

## Проверенные сценарии

Практически проверено полное удаление и восстановление:

- `frontend`;
- `backend01`;
- `backend02`;
- `db01`;
- `db02`;
- `monitoring`.

## Идемпотентность

Повторный запуск:

```bash
ansible-playbook playbooks/site.yml
```

для обычных конфигурационных задач не должен давать лишние изменения.

Ожидается:

```text
changed=0
failed=0
```

Исключение — задачи, которые специально создают новый backup, например backup GlusterFS metadata.

## Основные playbooks

| Playbook | Назначение |
|---|---|
| `playbooks/site.yml` | Полное развёртывание |
| `playbooks/recover-backend.yml` | Восстановление backend |
| `playbooks/recover-database.yml` | Восстановление db01/db02 |
| `playbooks/recover-monitoring.yml` | Восстановление monitoring |



## Результат

В результате создан веб-проект, который:

- автоматически разворачивается через Ansible;
- работает на нескольких VM;
- использует HTTPS;
- защищён `nftables`;
- балансирует web-трафик;
- продолжает работать при потере одного backend;
- использует MariaDB replication;
- использует GlusterFS;
- собирает метрики и логи;
- отправляет alerts;
- выполняет автоматические backup;
- позволяет восстановить полностью удалённые VM.

> **Ansible восстанавливает конфигурацию, а replication и backup позволяют восстановить данные и состояние инфраструктуры.**
