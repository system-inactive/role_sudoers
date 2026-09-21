# ansible-role-sms_sudoers

Тонкая обёртка над модулем [`community.general.sudoers`](https://docs.ansible.com/ansible/latest/collections/community/general/sudoers_module.html):
ставит пакет `sudo` и применяет список `sms_sudoers_rules` через этот модуль,
один вызов на элемент. Модуль сам проверяет синтаксис через `visudo` перед
записью каждого файла; роль дополнительно прогоняет `visudo -c` в конце —
проверку всей конфигурации целиком.

## Требования

- Ansible >= 2.12
- Коллекция `community.general` >= 13.1.0:

  ```bash
  ansible-galaxy collection install -r roles/sms_sudoers/requirements.yml
  ```

## Переменные

| Переменная | По умолчанию | Описание |
|---|---|---|
| `sms_sudoers_rules` | `[]` | Список правил — параметры `community.general.sudoers` как есть |
| `sms_sudoers_dir` | `/etc/sudoers.d` | `sudoers_path` модуля |
| `sms_sudoers_install_sudo` | `true` | Установить пакет `sudo`, если его нет |

Формат элемента `sms_sudoers_rules` — параметры модуля `community.general.sudoers`
без изменений: `name` (обязательно, это же имя файла), `user` или `group`,
`commands`, `state`, и необязательные `host`/`runas`/`nopassword`/`setenv`/
`noexec`/`defaults`. Роль подставляет `sudoers_path`, `validation: required`
и `nopassword: false` (у самого модуля по умолчанию `true`) — любой из них,
как и остальные параметры модуля, можно переопределить прямо в элементе.

## Пример

```yaml
- hosts: servers
  become: true
  roles:
    - role: sms_sudoers
      vars:
        sms_sudoers_rules:
          # Сменить свой пароль без запроса пароля sudo
          - name: alice-passwd
            user: alice
            runas: root
            nopassword: true
            commands: /usr/bin/passwd alice

          # Группа админов: sudo с паролем, кэш подтверждения на 5 минут
          - name: admins
            group: admins
            commands: ALL
            defaults:
              - "timestamp_timeout=5"

          # Удалить ранее заведённое временное правило
          - name: bob-passwd
            state: absent
```

## Тестирование

Роль покрыта сценарием [Molecule](https://ansible.readthedocs.io/projects/molecule/)
с Docker-драйвером. Сценарий `default` поднимает две системы — **Rocky Linux 9**
(RedHat) и **Ubuntu 24.04** (Debian):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r roles/sms_sudoers/molecule/default/requirements.yml

cd roles/sms_sudoers
molecule test
```

`molecule test` прогоняет роль дважды и падает при любом изменении на втором
прогоне — то есть роль дополнительно проверена на идемпотентность.

Статический анализ:

```bash
ansible-lint roles/sms_sudoers
```

## Лицензия

MIT
