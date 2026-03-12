# Infrastructure

Ansible-репозиторий для развёртывания CI/CD-инфраструктуры. Сейчас входит установка и настройка **Nexus Repository Manager 3** (OSS).

## Структура проекта

```
.
├── site.yml                    # Основной плейбук (роль Nexus)
├── inventory/
│   └── cicd/
│       ├── hosts.yml           # Инвентарь хостов
│       └── group_vars/
│           └── nexus.yml       # Переменные для группы nexus
├── templates/
│   ├── nexus.systemd.j2        # Unit systemd для сервиса Nexus
│   ├── nexus.vmoptions.j2      # Параметры JVM
│   └── nexus.properties.j2    # Конфигурация Nexus
└── README.md
```

## Требования

- **Ansible** (ansible-core 2.x)
- Доступ по **SSH** к целевым хостам (по ключу или паролю)
- Поддерживаемые ОС целевых хостов: **Debian/Ubuntu** или **RHEL/CentOS/Fedora**

Для Nexus 3.14 требуется **Java 8**; плейбук устанавливает OpenJDK 8 (на Debian — `openjdk-8-jdk`, на RedHat — `java-1.8.0-openjdk`).

## Подготовка

1. **Инвентарь** — в `inventory/cicd/hosts.yml` укажите хост и пользователя:
   - `ansible_host` — IP или hostname сервера;
   - `ansible_user` — пользователь для SSH.

2. **SSH** — ключ хоста должен быть в `~/.ssh/known_hosts` (один раз выполните `ssh user@host` и примите ключ или добавьте его через `ssh-keyscan`).

3. **Права** — пользователь на целевом хосте должен иметь возможность выполнять `sudo` без пароля или вы будете вводить пароль при запуске плейбука (`-K`).

## Запуск

```bash
ansible-playbook -i inventory/cicd/hosts.yml site.yml
```

Проверка подключения без применения изменений:

```bash
ansible -i inventory/cicd/hosts.yml nexus -m ping
```

## Настройка Nexus

Основные параметры задаются в `inventory/cicd/group_vars/nexus.yml`:

| Переменная | Описание |
|------------|----------|
| `nexus_version` | Версия Nexus (по умолчанию 3.14.0-04) |
| `nexus_port` | Порт веб-интерфейса (8081) |
| `nexus_java_home` | Путь к JDK 8 для systemd |
| `nexus_java_heap_size` | Размер heap JVM (1200M) |
| `nexus_ulimit` | Лимит nofile для пользователя nexus (65536) |
| `nexus_service_start_on_boot` | Запускать сервис при загрузке |

После первого запуска Nexus может подниматься несколько минут; веб-интерфейс: `http://<хост>:8081`.

---

## Внесённые изменения

Ниже перечислены правки, внесённые в плейбук и конфигурацию в процессе настройки.

### Зависимости и модули

- **Лимит nofile (pam_limits)** — модуль `community.general.pam_limits` заменён на встроенный `lineinfile`: в `/etc/security/limits.d/nexus.conf` записывается строка вида `nexus - nofile 65536`. Дополнительные коллекции Ansible не требуются.

### Установка JDK

- **Разные дистрибутивы** — установка JDK разведена по семействам ОС:
  - **RedHat** (RHEL/CentOS/Fedora): пакеты `java-1.8.0-openjdk`, `java-1.8.0-openjdk-devel`;
  - **Debian/Ubuntu**: пакет `openjdk-8-jdk` (Nexus 3.14 работает только с Java 8).

### Привилегии и владельцы (become_user)

- **Отказ от `become_user: nexus`** — при выполнении задач от непривилегированного пользователя Ansible на части хостов пытался выставить права на временные файлы через режим в стиле ACL (`A+user:nexus:rx:allow`), что приводило к ошибке `chmod: invalid mode`. Все задачи переписаны так, чтобы выполняться от root (`become: true`) с явным указанием владельца и группы через параметры модулей (`owner`, `group`) или отдельными задачами `file`. Затронуты: загрузка и распаковка Nexus, создание симлинка, правка `.bashrc` и `nexus.rc`, шаблоны `nexus.vmoptions`, `nexus.properties`, правка `system.properties`.

### Сервис systemd (Nexus)

- **JAVA_HOME и INSTALL4J_JAVA_HOME** — в unit добавлены переменные окружения `JAVA_HOME` и `INSTALL4J_JAVA_HOME` (лаунчер Nexus ищет JVM по `INSTALL4J_JAVA_HOME`). Значение задаётся переменной `nexus_java_home` в `group_vars/nexus.yml`.
- **WorkingDirectory** — для сервиса указана рабочая директория `nexus_directory_home`, чтобы скрипт старта выполнялся в нужном каталоге.
- **Путь к Java 8** — по умолчанию для Debian/Ubuntu задано `nexus_java_home: "/usr/lib/jvm/java-8-openjdk-amd64"`; для RHEL при необходимости можно переопределить в инвентаре (например, каталог пакета `java-1.8.0-openjdk`).

### Инвентарь

- Отключена проверка ключа хоста `ansible_ssh_common_args: '-o StrictHostKeyChecking=no'` (только для тестовых окружений).
