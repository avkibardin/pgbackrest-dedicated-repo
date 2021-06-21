# Настройка резервного копирования кластеров PostgreSQL с помощью pgbackres.

## 1. Предварительные замечания.

Утилита pgbackrest позволяет создавать файловые копии кластеров PostgreSQL с поддержкой полных, дифференциальных и инкрементальных бэкапов. 

**Полный бекап** включает в себя полную копию кластера, плюс wal файлы, созданные во время создания резервной копии.

**Дифференциальные бекапы** включают в себя изменения относительно полного бэкапа, к которому они привязаны.

**Инкрементальные бекапы** включают в себя изменения относительно последнего бэкапа, тип бэкапа при этом не имеет значения. Если полного бэкапа не существует, то вместо инкрементального будет создан полный бэкап.

По умолчанию, если явно не указан тип бэкапа, pgbackrest попытается создать инкрементальный бэкап. Кроме того, явно указать сколько делать инкрементальных бэкапов между полными бэкапами нельзя, всё это регулируется на основе заданий в кроне. Все устаревшие бекапы будут автоматически удалены самим pgbackrest.

Для Ubuntu имеются сборки, включенные в официальный репозитарий PostgreSQL:

https://www.postgresql.org/download/linux/ubuntu/

Установку для RPM-based  дистрибутивов можно почитать тут:

https://pgbackrest.org/user-guide-rhel.html

По сути будет отличаться только установка требуемых пакетов.

Официальная документация доступна тут:
https://pgbackrest.org/user-guide.html

Подробное описание параметров конфигурации:
https://pgbackrest.org/configuration.html

## 2. Описание демо стенда

В данной инструкции будет рассмотрена общая установка из исходных кодов. План резервного копирования будет такой:

- каждую ночь в 2:00 создавать полный бэкап
- в будние дни, каждый час с 9:00 до 18:00 создавать инкрементальный бэкап
- резервные копии складировать на отдельном сервере, так называемый *репозиторий*, в терминологии pgbackrest
- максимальное кол-во полных бэкапов - 2.

Демо стенд:

| server     | OS           | ip        |
| ---------- | ------------ | --------- |
| repository | Ubuntu 20.04 | 10.0.2.13 |
| PostgreSQL | Ubuntu 20.04 | 10.0.2.14 |

Директории на сервере БД:

- конфигурационные файлы кластера: /etc/postgresql/12/main
- файлы кластера: /var/lib/postgresql/12/main

Директория хранения резервных копий на сервере репозитория: /var/lib/pgbackrest.

**Важно!** Для корректной работы версии и архитектуры pgbackrest на всех серверах должны быть идентичными.

Все дальнейшие действия выполняются от пользователя root, если не оговорено отдельно.

## 3. Настройка сервера баз данных.

Скачать исходники

```bash
wget https://github.com/pgbackrest/pgbackrest/archive/release/2.34.tar.gz
```

Распаковать

```bash
tar -xzf 2.34.tar.gz
```

Установить пакеты, необходимые для сборки pgbackrest:

```bash
apt update && apt install -y make gcc libpq-dev libssl-dev libxml2-dev pkg-config \
      liblz4-dev libzstd-dev libbz2-dev libz-dev
```

Собрать бинарник:

```bash
cd pgbackrest-release-2.34/src && ./configure && make
```

На выходе получится один бинарный файл pgbackrest.

Скопировать файл 

```bash
cp pgbackrest /usr/local/bin/pgbackrest
```

Установить дополнительные пакеты, необходимые для работы pgbackrest

```bash
apt update && apt install -y postgresql-client libxml2
```

Создать папки, файлы и раскидать права:

```bash
mkdir -p -m 770 /var/log/pgbackrest
chown postgres:postgres /var/log/pgbackrest
mkdir -p /etc/pgbackrest
mkdir -p /etc/pgbackrest/conf.d
touch /etc/pgbackrest/pgbackrest.conf
chmod 640 /etc/pgbackrest/pgbackrest.conf
chown postgres:postgres /etc/pgbackrest/pgbackrest.conf
```

Проверить версию

```bash
sudo -u postgres pgbackrest
```

## 4. Настройка PostgreSQL и конфигурации pgbackrest на сервере баз данных.

Включить прослушку по всем интерфейсам в /etc/postgresql/12/main/postgresql.conf:

```
listen_addresses = '*'
```

Добавить настройки для резервного копирования:

```
archive_command = 'pgbackrest --stanza=demo archive-push %p'
archive_mode = on
listen_addresses = '*'
log_filename = 'postgresql.log'
log_line_prefix = ''
max_wal_senders = 3
wal_level = replica
```

После чего необходимо перезапустить кластер

```bash
pg_ctlcluster 12 main restart
```

Настроить pgbackrest на стороне сервера баз данных. Для этого в файл /etc/pgbackrest/pgbackrest.conf внести строки:

```
[demo]
pg1-path=/var/lib/postgresql/12/main

[global]
log-level-file=detail
repo1-host=10.0.2.13
```

## 5. Настройка сервера репозитариев

Скопировать бинарный файл pgbackrest с сервера баз данных, либо выполнить сборку аналогично пункту 2, до этапа создания директорий и файлов конфигурации.

Далее необходимо будет создать отдельного пользователя и назначить его владельцем для директорий и файлов.

Создание пользователя:

```bash
adduser --disabled-password --gecos "" pgbackrest
```

Установить  пакеты для работы pgfbackrest:

```bash
apt-get install postgresql-client libxml2
```

Создать директории и файлы, раздать права:

```bash
mkdir -p -m 770 /var/log/pgbackrest
chown pgbackrest:pgbackrest /var/log/pgbackrest
mkdir -p /etc/pgbackrest
mkdir -p /etc/pgbackrest/conf.d
touch /etc/pgbackrest/pgbackrest.conf
chmod 640 /etc/pgbackrest/pgbackrest.conf
chown pgbackrest:pgbackrest /etc/pgbackrest/pgbackrest.conf
```

Создать директорию для хранения репозитория резервных копий:

```bash
mkdir -p /var/lib/pgbackrest
chmod 750 /var/lib/pgbackrest
chown pgbackrest:pgbackrest /var/lib/pgbackrest
```

Проверить запуск pgbackrest:

```bash
sudo -u pgbackrest pgbackrest
```

Настроить сервер репозитариев, для этого внести /etc/pgbackrest/pgbackrest.conf строки:

```
[demo]
pg1-host=10.0.2.14
pg1-path=/var/lib/postgresql/12/main

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full-type=count
repo1-retention-full=2
start-fast=y
stop-auto=y

[global:archive-push]
compress-level=3
```

## 6. Настройка авторизации ключами по ssh

Необходимо настроить авторизацию открытыми ключами. С сервера репозитория пользователь pgbackrest должен авторизоваться на севре базданных под пользователем postgres. С сервера баз данных, пользователь postgres должен иметь право авторизоваться на сервер баз данных под пользователем pgbackrest.

На сервер баз данных создайте  директорию для хранения ключей и сгенерируйте ключи для пользователя postgres:

```bash
sudo -u postgres mkdir -m 750 -p /var/lib/postgresql/.ssh
sudo -u postgres ssh-keygen -f /var/lib/postgresql/.ssh/id_rsa -t rsa -b 4096 -N ""
```

Аналогично настроить на сервере репозитория для пользователя pgbackrest:

```bash
sudo -u pgbackrest mkdir -m 750 /home/pgbackrest/.ssh
sudo -u pgbackrest ssh-keygen -f /home/pgbackrest/.ssh/id_rsa -t rsa -b 4096 -N ""
```

Далее необходимо обменяться ключами. Для этого на сервере postgresql выполнить:

```bash
(echo -n 'no-agent-forwarding,no-X11-forwarding,no-port-forwarding,' && \
       echo -n 'command="/usr/local/bin/pgbackrest ${SSH_ORIGINAL_COMMAND#* }" ' && \
       sudo ssh root@10.0.2.13 cat /home/pgbackrest/.ssh/id_rsa.pub) | \
       sudo -u postgres tee -a /var/lib/postgresql/.ssh/authorized_keys
```

На сервере репозитория выполнить:

```bash
(echo -n 'no-agent-forwarding,no-X11-forwarding,no-port-forwarding,' && \
       echo -n 'command="/usr/local/bin/pgbackrest ${SSH_ORIGINAL_COMMAND#* }" ' && \
       sudo ssh root@10.0.2.14 cat /var/lib/postgresql/.ssh/id_rsa.pub) | \
       sudo -u pgbackrest tee -a /home/pgbackrest/.ssh/authorized_keys
```

Либо просто скопировать содержимое  публичных ключей в соответствующие файлы authorized_keys

Проверить подключения. С сервера postgresql:

```bash
sudo -u postgres ssh pgbackrest@10.0.2.13
```

С сервера репозитория:

```bash
sudo -u pgbackrest ssh postgres@10.0.2.14
```

## 7. Инициализация и запуск резервного копирования.

Создать конфигурацию резервного копирования кластера (stanza) на сервере репозитория:

```bash
sudo -u pgbackrest pgbackrest --stanza=demo stanza-create
```

Проверить подключение на сервере репозитория

```bash
sudo -u pgbackrest pgbackrest --stanza=demo --log-level-console=info check
```

Вывод должен быть примерно таким:

```
2021-06-21 10:21:47.797 P00   INFO: check command begin 2.34: --exec-id=188524-b85c5145 --log-level-console=info --pg1-host=10.0.2.14 --pg1-path=/var/lib/postgresql/12/main --repo1-path=/var/lib/pgbackrest --stanza=demo
2021-06-21 10:21:48.915 P00   INFO: check repo1 configuration (primary)
2021-06-21 10:21:49.118 P00   INFO: check repo1 archive for WAL (primary)
2021-06-21 10:21:49.825 P00   INFO: WAL segment 000000010000000000000008 successfully archived to '/var/lib/pgbackrest/archive/demo/12-1/0000000100000000/000000010000000000000008-3d5eb7f36e92508f1ddfb0e076ed87dae0f58d7d.gz' on repo1
2021-06-21 10:21:49.927 P00   INFO: check command end: completed successfully (2130ms)
```

Проверить подключение на сервере PostgreSQL:

```bash
sudo -u postgres pgbackrest --stanza=main1 --log-level-console=info check
```

Вывод должен быть примерно таким:

```
2021-06-21 10:22:02.414 P00   INFO: check command begin 2.34: --exec-id=184720-12f209aa --log-level-console=info --log-level-file=detail --pg1-path=/var/lib/postgresql/12/main --repo1-host=10.0.2.13 --stanza=demo
2021-06-21 10:22:02.820 P00   INFO: check repo1 configuration (primary)
2021-06-21 10:22:03.497 P00   INFO: check repo1 archive for WAL (primary)
2021-06-21 10:22:04.203 P00   INFO: WAL segment 000000010000000000000009 successfully archived to '/var/lib/pgbackrest/archive/demo/12-1/0000000100000000/000000010000000000000009-88d92dca23c409d6121658bb74623e818ed69519.gz' on repo1
2021-06-21 10:22:04.303 P00   INFO: check command end: completed successfully (1890ms)
```

Запустить тестовое создание резервной копии на стороне сервера репозитария:

```bash
sudo -u pgbackrest pgbackrest --stanza=demo  --log-level-console=info --type=full backup
```

Проверить состояние резервных копий можно командой:

```bash
sudo -u pgbackrest pgbackrest info
```

## 8. Планирование задания в кроне

Напомним, что план резервного копирования будет такой:

- каждую ночь в 2:00 создавать полный бэкап
- в будние дни, каждый час с 9:00 до 18:00 создавать инкрементальный бэкап

Для этого  на сервер репозитария создать файл /etc/cron.d/pgbackrest-test со следующим содержимым:

```
#
#Задания для создания бекапов
#
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# в будни каждый час с 9:00 до 18:00 делать инкрементальный бэкап
0 9-18       * * 1-5 pgbackrest pgbackrest --stanza=demo --type=incr backup

# каждую ночь делать полный бэкап
0 2     * * *   pgbackrest pgbackrest --stanza=demo --type=full  backup
```

## 9. Восстановление из резервной копии.

Для воcстановления из резервной копии все команды надо будет выполнять на стороне сервера баз данных.

Для полного восстановления кластера, необходимо предварительно остановить кластер. Удалить всё содержимое с данными кластера, очистив директорию /var/lib/postgresql/12/main. Затем выполнить команды:

```bash
sudo -u postgres pgbackrest --stanza=demo restore
sudo pg_ctlcluster 12 demo start
```

Либо можно восстанавливать только изменившиеся файлы, что удобно в случае битых файлов кластера. В данном варианте pgbackrest будет проверять файлов по sha-1, и файлы, которые отличаются от файлов в репозитории будет восстанавливать из резервной копии. Файлы, отсутствующие в резервной копии будут удалены. Для запуска такого восстановления, предварительно остановите кластер, если он запущен, и затем выполните команды:

```bash
sudo -u postgres pgbackrest --stanza=demo --delta -log-level-console=detail restore
sudo pg_ctlcluster 12 demo start
```

