# Ansible Role: vault-anycast

Эта Ansible роль автоматизирует развёртывание динамического анонса anycast‑адреса для кластера HashiCorp Vault с использованием демона маршрутизации BIRD и протокола OSPF.

## Задача

При работе кластера Vault требуется, чтобы anycast‑адрес (например, 172.24.64.64) анонсировался в OSPF‑домене только с тех узлов, которые в данный момент готовы принимать трафик.

Решение основано на:

- постоянном присутствии anycast‑адреса на loopback‑интерфейсе (для работы Vault);
- статическом маршруте в BIRD с next‑hop через физический интерфейс;
- экспорте статического маршрута в OSPF как внешнего LSA (Type 5);
- управлении включением/выключением статического протокола через birdc по сигналу health‑check скрипта;
- исключении loopback из OSPF для предотвращения автоматической генерации stub‑сетей.

Роль настраивает BIRD, health‑check скрипт и systemd‑таймер, а также добавляет зависимости в bird.service от vault.service.

## Требования

+ Целевая ОС: Ubuntu 20.04/22.04, Debian 11/12, RHEL/CentOS 8/9 (или аналогичные с поддержкой BIRD 2.x).
+ Ansible: версия 2.9 или выше.
+ Vault: должен быть предварительно установлен и настроен (роль не управляет установкой Vault).
+ Права: для выполнения задач требуются привилегии become: yes.

## Переменные

Все переменные имеют значения по умолчанию в defaults/main.yml. Их можно переопределить в инвентори, в плейбуке или в group_vars.

|Переменная|Описание|Значение по умолчанию|
|----------|--------|--------------------:|
|anycast_ip|Anycast‑адрес в формате X.X.X.X/32|172.24.64.64/32|
|vault_api_url|URL для health‑check Vault|https://127.0.0.1:8200|
|vault_health_codes_healthy|Список HTTP‑кодов, при которых Vault считается здоровым| 200, 429|
|healthcheck_interval_sec|Интервал запуска health‑check скрипта (сек)| 30 |
|bird_router_id | Router ID для BIRD (IPv4‑адрес). Обязательно уникальный для каждого узла | {{ansible_default_ipv4.address}}|
| bird_ospf_interface | Имя физического интерфейса, на котором работает OSPF | ens34 |
|bird_ospf_cost | Метрика OSPF для интерфейса |10|
|bird_ospf_auth_algorithm | Алгоритм аутентификации OSPF | hmac sha256 |
|bird_ospf_key_id |ID ключа аутентификации | 1|
|bird_ospf_password | Пароль для аутентификации OSPF (обязательно переопределить!)| |
|configure_loopback_address | Добавлять anycast IP на loopback? | true |
|manage_anycast_protocol | Управлять статическим протоколом через скрипт| true|

## Пример использования

### Инвентори (inventory/production.ini)

```ini
[vault_servers]
vault1 ansible_host=172.24.68.4 bird_router_id=172.24.68.4
vault2 ansible_host=172.24.68.5 bird_router_id=172.24.68.5
vault3 ansible_host=172.24.68.6 bird_router_id=172.24.68.6
```

### Плейбук (deploy.yml)
```yaml
- hosts: vault_servers
  become: yes
  vars:
    bird_ospf_password: "3DF251949166B5DC63D8B1BC49F2FF19"
  roles:
    - ansible-role-vault-anycast
```

## Запуск

```bash
ansible-playbook -i inventory/production.ini deploy.yml
```

## Что делает роль

1. Устанавливает пакет bird2 (BIRD 2.x).
2. Копирует health‑check скрипт /usr/local/bin/vault-healthcheck.sh и делает его исполняемым.
3. Создаёт systemd unit для скрипта (vault-healthcheck.service) и таймер (vault-healthcheck.timer) с заданным интервалом.
4. Добавляет anycast IP на loopback (опционально).
5. Генерирует конфигурацию BIRD из шаблона и проверяет её синтаксис.
6. Создаёт override для bird.service, добавляя зависимости After=vault.service и Wants=vault.service.
7. Включает и запускает сервисы bird и vault-healthcheck.timer.

## Проверка работы

После применения роли:

- На каждом узле убедитесь, что OSPF‑соседства установлены:

    ```bash
    birdc show ospf neighbor
    ```

- Проверьте, что статический протокол включён (если Vault здоров):

    ```bash
    birdc show protocols static_vault_anycast
    ```

- На маршрутизаторе (MikroTik, Cisco и т.д.) проверьте наличие external LSA и маршрута в таблице.

## Примечания

- Роль не управляет установкой или конфигурацией Vault. Предполагается, что Vault уже установлен и его сервис называется vault.service.

- Аутентификация OSPF включена по умолчанию. Если аутентификация не нужна, установите bird_ospf_password: "" и закомментируйте соответствующий блок в шаблоне bird.conf.j2.

- Если на интерфейсе lo уже есть anycast IP, скрипт не будет его добавлять повторно.

- Для сохранения anycast IP после перезагрузки рекомендуется добавить его в конфигурацию сети (например, в netplan или /etc/network/interfaces). Роль делает попытку добавить его в netplan, но лучше настраивать отдельно.

## Лицензия
MIT

## Автор

(C) 2026 Oleg Yakovlev <yakovlev.oleg@gmail.com>, [https://net4you.ru](https://net4you.ru). All rights reserved

Исходный текст роли: https://github.com/Simargl21/ansible-role-vault-anycast 
