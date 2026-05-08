# Тестовое задание DevOps

## Назначение проекта

Проект предназначен для автоматизации настройки виртуальной машины Debian 12 с помощью Ansible.

На текущем этапе выполнена базовая подготовка окружения:

- создана виртуальная машина Debian 12 в VirtualBox;
- настроен SSH-доступ к виртуальной машине;
- настроен вход по SSH-ключу;
- установлен Ansible в WSL;
- создана структура Ansible-проекта;
- проверено управление виртуальной машиной через `ansible ping`.

## Окружение

- Основная ОС: Windows 10
- Среда для Ansible: WSL Ubuntu 20.04
- Виртуализация: VirtualBox
- Управляемая машина: Debian 12
- Пользователь на Debian VM: `devops`
- Доступ к VM: SSH по ключу
- SSH-ключ: `~/.ssh/devops_vm`

## Структура проекта

```text
devops-test/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── playbooks/
│   └── ping.yml
├── scripts/
│   └── ansible_ping.sh
├── group_vars/
└── roles/



