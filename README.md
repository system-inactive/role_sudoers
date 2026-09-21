# ansible-role-sms_sudoers

Управляет drop-in файлами правил в `/etc/sudoers.d`. Каждый файл перед
записью проверяется через `visudo -cf`, поэтому синтаксическая ошибка в
правиле не попадёт на хост и не сломает `sudo`. В конце роль ещё раз
прогоняет `visudo -c` — проверку всей конфигурации целиком.

Роль **не трогает** основной `/etc/sudoers` — только файлы в
`/etc/sudoers.d`, имена которых начинаются с `sms_sudoers_prefix`
(по умолчанию `ansible-`). Чужие файлы в каталоге не затрагиваются.

## Требования

- Ansible >= 2.12
- На целевом хосте доступен `visudo` (ставится вместе с пакетом `sudo`;
  роль умеет установить его сама — см. `sms_sudoers_install_sudo`)

## Поддерживаемые ОС

| Семейство | Дистрибутивы | Покрыто тестами |
|---|---|---|
| Debian | Debian 12/13, Ubuntu 22.04/24.04 | Ubuntu 24.04 |
| RedHat | RHEL/Rocky/AlmaLinux 8, 9 | Rocky Linux 9 |

Роль дистрибутивонезависима (использует только модули `package`, `template`,
`file` и `command`), список выше отражает то, на чём она реально прогоняется в
Molecule.

## Переменные роли

Все переменные и их значения по умолчанию — в [`defaults/main.yml`](defaults/main.yml).

| Переменная | По умолчанию | Описание |
|---|---|---|
| `sms_sudoers_rules` | `[]` | Список файлов с правилами (формат ниже) |
| `sms_sudoers_passwd_users` | `[]` | Список пользователей, которым нужно только сменить свой пароль без запроса пароля sudo (см. ниже) |
| `sms_sudoers_dir` | `/etc/sudoers.d` | Каталог drop-in файлов |
| `sms_sudoers_prefix` | `ansible-` | Префикс имён файлов, которыми управляет роль |
| `sms_sudoers_purge` | `false` | Удалять файлы с этим префиксом, которых нет в `sms_sudoers_rules` |
| `sms_sudoers_install_sudo` | `true` | Установить пакет `sudo`, если его нет |

### Формат элемента `sms_sudoers_rules`

Каждый элемент описывает один файл в `sms_sudoers_dir`:

| Поле | По умолчанию | Описание |
|---|---|---|
| `name` | — (обязательно) | Имя файла без префикса. Только `A-Z a-z 0-9 _ -` (sudo игнорирует файлы с точкой или `~` в имени) |
| `state` | `present` | `present` — создать/обновить, `absent` — удалить файл |
| `rules` | `[]` | Список правил (см. ниже). Для `present` нужно хотя бы одно из `rules`/`lines` |
| `lines` | `[]` | Произвольные строки, попадают в файл как есть (например, `Defaults:...`) |

Поля записи внутри `rules`:

| Поле | По умолчанию | Описание |
|---|---|---|
| `who` | — (обязательно) | Пользователь, `%группа` или список таких значений |
| `hosts` | `ALL` | Значение поля host в правиле |
| `runas` | `ALL` | Кем разрешено выполнять (`(runas)`) |
| `nopasswd` | `false` | `true` — добавить тег `NOPASSWD:` |
| `tags` | `[]` | Дополнительные теги (`SETENV`, `NOEXEC`, …) |
| `commands` | `ALL` | Команда или список команд |

Роль проверяет входные данные ещё до записи: корректность `name`, значение
`state`, наличие `rules`/`lines` у `present`-правил и непустой `who` у каждой
записи `rules`. При ошибке выполнение падает на `assert` с понятным
сообщением, до того как что-либо попадёт на хост.

## Пример плейбука

```yaml
- hosts: servers
  become: true
  roles:
    - role: sms_sudoers
      vars:
        sms_sudoers_rules:
          # Сменить свой пароль без запроса пароля sudo
          - name: alice-passwd
            rules:
              - who: alice
                runas: root
                nopasswd: true
                commands: /usr/bin/passwd alice

          # Деплою можно перезапускать nginx без пароля
          - name: deploy-nginx
            rules:
              - who: deploy
                nopasswd: true
                commands:
                  - /usr/bin/systemctl restart nginx
                  - /usr/bin/systemctl reload nginx

          # Группа админов: sudo с паролем, кэш подтверждения на 5 минут
          - name: admins
            lines:
              - "Defaults:%admins timestamp_timeout=5"
            rules:
              - who: "%admins"

          # Удалить ранее заведённое временное правило
          - name: bob-passwd
            state: absent
```

Файл `alice-passwd` окажется по пути `/etc/sudoers.d/ansible-alice-passwd`
с содержимым:

```
alice ALL=(root) NOPASSWD: /usr/bin/passwd alice
```

## Purge

Со `sms_sudoers_purge: true` роль удаляет из `sms_sudoers_dir` все файлы с
префиксом `sms_sudoers_prefix`, которых нет в `sms_sudoers_rules`. Это делает
набор правил роли декларативным: убрали элемент из списка — файл исчез с хоста.
Файлы без этого префикса (заведённые вручную или другими ролями) не
затрагиваются, поэтому пустой `sms_sudoers_prefix` вместе с `purge` запрещён
проверкой.

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

`converge` разворачивает набор правил (present, `lines`, список команд,
purge), а также заранее оставляет «осиротевший» файл с префиксом роли. `verify`
проверяет: файлы созданы с режимом `0440`, их содержимое совпадает с ожидаемым,
`state: absent` и purge удалили нужные файлы, а `visudo -c` проходит без
ошибок. `molecule test` дополнительно прогоняет роль дважды и падает при любом
изменении на втором прогоне — то есть роль проверена на идемпотентность.

Статический анализ:

```bash
ansible-lint roles/sms_sudoers
```

## Лицензия

MIT
