# Тестовое задание DevOps

## Назначение проекта

Проект содержит Ansible-сценарии для настройки виртуальной машины Debian 12 в VirtualBox и развёртывания Gitea.

Основные выполненные задачи:

- создана виртуальная машина Debian 12 в VirtualBox;
- настроен SSH-доступ к VM по ключу;
- Ansible установлен на хосте через WSL;
- реализован playbook установки Docker;
- реализована установка Gitea как бинарника;
- Gitea настроена без SSH и без HTTPS;
- база данных Gitea — SQLite;
- директория репозиториев Gitea размещена в `/var/lib/gitea/repositories`;
- реализованы playbook-и резервного копирования, восстановления и обновления Gitea.

## Окружение

- Host OS: Windows 10
- Ansible control node: WSL Ubuntu
- Managed node: Debian 12 в VirtualBox
- Подключение к VM: SSH по ключу
- Пользователь на VM: `devops`

## Структура проекта

```text
foton-test-task/
├── README.md
├── ansible.cfg
├── inventory.ini
├── group_vars/
│   └── all.yml
├── playbooks/
│   ├── 00_ping.yml
│   ├── 01_install_docker.yml
│   ├── 02_install_gitea.yml
│   ├── 03_backup_gitea.yml
│   ├── 04_restore_gitea.yml
│   └── 05_update_gitea.yml
├── templates/
│   ├── app.ini.j2
│   └── gitea.service.j2
└── backups/
    └── .gitkeep
```

## Важное замечание по запуску

Команды Ansible нужно выполнять из корня проекта:

```bash
cd ~/foton-test-task
```

Если запускать Ansible из другой директории, он может не найти `ansible.cfg` и `inventory.ini`.

## Inventory

Файл inventory:

```text
inventory.ini
```

В нём описана управляемая Debian VM.

Пример проверки доступности VM:

```bash
ansible vm -m ping
```

Ожидаемый результат:

```text
debian12 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Проверка SSH-доступа

SSH-доступ к VM выполняется по ключу:

```bash
ssh -i ~/.ssh/devops_vm devops@<vm_ip>
```

IP-адрес VM указан в `inventory.ini`.

Если IP изменился после перезагрузки VM или роутера, его можно проверить внутри Debian VM командой:

```bash
hostname -I
```

После этого нужно обновить `ansible_host` в `inventory.ini`.

## Playbook 00: проверка подключения

```bash
ansible-playbook playbooks/00_ping.yml
```

Назначение:

- проверяет, что Ansible подключается к Debian VM;
- использует SSH-доступ по ключу;
- возвращает `pong`.

## Playbook 01: установка Docker

```bash
ansible-playbook playbooks/01_install_docker.yml -K
```

Назначение:

- добавляет официальный Docker apt-репозиторий;
- устанавливает Docker Engine;
- устанавливает Docker Compose plugin;
- включает и запускает сервис Docker;
- добавляет пользователя `devops` в группу `docker`.

Проверка после установки:

```bash
ansible vm -m command -a "systemctl is-active docker"
ansible vm -m command -a "docker --version"
ansible vm -m command -a "docker compose version"
ansible vm -m command -a "docker ps"
```

## Playbook 02: установка Gitea

```bash
ansible-playbook playbooks/02_install_gitea.yml -K
```

Назначение:

- создаёт системного пользователя `git`;
- создаёт рабочие директории Gitea;
- скачивает бинарник Gitea;
- создаёт конфигурационный файл `/etc/gitea/app.ini`;
- создаёт systemd unit;
- запускает сервис Gitea.

Основные параметры Gitea:

```text
Бинарник: /usr/local/bin/gitea
Конфигурация: /etc/gitea/app.ini
Рабочая директория: /var/lib/gitea
Репозитории: /var/lib/gitea/repositories
База данных: SQLite
HTTP порт: 3000
SSH Gitea: отключён
HTTPS: не используется
```

Проверка:

```bash
ansible vm -m command -a "systemctl is-active gitea"
ansible vm -m uri -a "url=http://127.0.0.1:3000 status_code=200"
```

## Playbook 03: резервное копирование Gitea

```bash
ansible-playbook playbooks/03_backup_gitea.yml -K
```

Назначение:

- останавливает Gitea перед созданием backup;
- выполняет `gitea dump`;
- запускает Gitea после backup;
- скачивает архив backup в локальную директорию `backups/`.

Формат команды dump:

```bash
gitea dump --verbose -f "<выходной архив дампа>.zip" -w /var/lib/gitea/ -c /etc/gitea/app.ini
```

Архивы backup сохраняются локально в:

```text
backups/
```

Реальные `.zip`-архивы не добавляются в Git. В репозитории хранится только файл:

```text
backups/.gitkeep
```

## Playbook 04: восстановление Gitea из backup

Восстановление является разрушительной операцией: текущие данные Gitea будут заменены данными из backup.

Пример запуска:

```bash
ansible-playbook playbooks/04_restore_gitea.yml -K -e "backup_file=backups/gitea-dump-YYYYMMDDTHHMMSS.zip confirm_restore=true"
```

Назначение:

- проверяет наличие локального backup-архива;
- загружает backup на VM;
- распаковывает архив;
- восстанавливает `app.ini`;
- восстанавливает данные Gitea;
- восстанавливает SQLite-базу из `gitea-db.sql`;
- восстанавливает репозитории, если они есть в архиве;
- запускает Gitea;
- проверяет HTTP endpoint.

Защита от случайного запуска:

```text
confirm_restore=true
```

Без этого параметра playbook завершится ошибкой.

## Playbook 05: обновление Gitea

```bash
ansible-playbook playbooks/05_update_gitea.yml -K
```

Назначение:

- проверяет текущую версию Gitea;
- скачивает целевой бинарник;
- останавливает сервис Gitea;
- сохраняет backup текущего бинарника;
- заменяет бинарник;
- запускает Gitea;
- проверяет HTTP endpoint;
- выводит новую версию.

Для обновления до конкретной версии можно передать переменную:

```bash
ansible-playbook playbooks/05_update_gitea.yml -K -e "target_gitea_version=1.26.1"
```

## Проверка качества

Для Ansible playbook-ов используется `ansible-lint`.

Проверка всех playbook-ов:

```bash
ansible-lint playbooks/00_ping.yml
ansible-lint playbooks/01_install_docker.yml
ansible-lint playbooks/02_install_gitea.yml
ansible-lint playbooks/03_backup_gitea.yml
ansible-lint playbooks/04_restore_gitea.yml
ansible-lint playbooks/05_update_gitea.yml
```

Для bash-скриптов используется `shellcheck`.

Проверка:

```bash
shellcheck scripts/*.sh
```

Если директории `scripts/` нет или bash-скрипты не используются, эта проверка не требуется.

## Порядок полного запуска

Рекомендуемый порядок:

```bash
cd ~/foton-test-task

ansible-playbook playbooks/00_ping.yml
ansible-playbook playbooks/01_install_docker.yml -K
ansible-playbook playbooks/02_install_gitea.yml -K
ansible-playbook playbooks/03_backup_gitea.yml -K
ansible-playbook playbooks/04_restore_gitea.yml -K -e "backup_file=backups/<backup>.zip confirm_restore=true"
ansible-playbook playbooks/05_update_gitea.yml -K
```

## Git

Backup-архивы не должны попадать в репозиторий.

Проверка состояния:

```bash
git status
```

Перед публикацией на GitHub рабочее дерево должно быть чистым:

```text
nothing to commit, working tree clean
```
