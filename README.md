# Проектная работа по теме "Реализация веб-проекта с использованием оркестрации Ansible и автоматическим восстановлением после сбоя".

## Цели проектной работы:

- закрепить и продемонстрировать полученные знания и навыки;
- создать веб-проект;
- подготовить проект для профессионального портфолио.

По требованиям курса веб-проект с несколькими виртуальными машинами должен включать:

- включен https;
- сновная инфраструктура в DMZ зоне;
- firewall на входе;
- сбор метрик и настроенный алертинг;
- организован централизованный сбор логов;
- организован backup.

В моей реализации дополнительно реализованы отказоустойчивость web-части, MariaDB Primary/Replica, GlusterFS и автоматизированное восстановление после полного удаления отдельных VM.


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


## Архитектура

Проект состоит из шести виртуальных машин:
```
| VM | Назначение |
|---|---|
| `frontend` | Nginx, HTTPS, reverse proxy, балансировка |
| `backend01` | Apache + PHP + WordPress, GlusterFS brick |
| `backend02` | Apache + PHP + WordPress, GlusterFS brick |
| `db01` | MariaDB PRIMARY |
| `db02` | MariaDB REPLICA |
| `monitoring` | Prometheus, Grafana, Loki, Alertmanager, Blackbox Exporter, GlusterFS arbiter |
```

Сети разделены по назначению:

- management network — управление VM;
- application network — frontend ↔ backend;
- database network — backend ↔ MariaDB.

```text
                    HTTPS
                      |
                      v
                 frontend
                  Nginx
                 /     \
                v       v
         backend01     backend02
       Apache/WordPress Apache/WordPress
                \       /
                 \     /
                  GlusterFS
                Replica 2
                 + Arbiter

           backend01/02
                |
                v
        db01 PRIMARY -> db02 REPLICA

          monitoring
 Prometheus / Grafana / Loki
 Alertmanager / Blackbox / Alloy
```

## Соответствие требованиям OTUS
```
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
```



## Развёртывание

## Запуск проекта

Запустить виртуальные машины:

```bash
vagrant up
```

Проверить доступность:

```bash
ansible all -m ping
```

Развернуть инфраструктуру:

```bash
ansible-playbook playbooks/site.yml
```

Для доступа к сайту добавить:

```text
192.168.56.10 project.local
```

Сайт:

```text
https://project.local
```

## Восстановление после полного удаления VM

### Frontend

```bash
vagrant destroy -f frontend
vagrant up frontend
ansible frontend -m ping
ansible-playbook playbooks/site.yml --limit frontend
```

### Backend

```bash
vagrant destroy -f backend01
vagrant up backend01
ansible backend01 -m ping
ansible-playbook playbooks/recover-backend.yml --limit backend01
```

### Database

```bash
vagrant destroy -f db02
vagrant up db02
ansible db02 -m ping
ansible-playbook playbooks/recover-database.yml --limit db02
```

### Monitoring

```bash
vagrant destroy -f monitoring
vagrant up monitoring
ansible monitoring -m ping
ansible-playbook playbooks/recover-monitoring.yml --limit monitoring
```

На практике проверено полное удаление и восстановление:

- `frontend`
- `backend01`
- `backend02`
- `db01`
- `db02`
- `monitoring`

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
