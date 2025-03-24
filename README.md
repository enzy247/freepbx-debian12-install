# Установка FreePBX 17 на Debian 12

Подробная инструкция по установке FreePBX 17 на Debian 12.

## 1. Подготовка системы

### Вход в систему и обновление пакетов
```bash
sudo su -
apt-get update && apt-get upgrade -y
```

### Установка необходимых зависимостей
```bash
apt -y install build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev uuid-dev default-libmysqlclient-dev htop sngrep lame ffmpeg mpg123
apt -y install git vim curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev build-essential libjansson-dev libxml2-dev uuid-dev expect
```

## 2. Установка PHP 8.3

# Добавление нового PGP-ключа и обновление репозитория

## Добавление нового ключа PGP

Автор репозитория сменил PGP-ключи. Для загрузки нового ключа рекомендуется использовать анонимайзер, прокси или VPN.

### Способы скачивания ключа:

1. **Прямая загрузка** (требуется VPN/прокси):
   ```
   https://packages.sury.org/php/apt.gpg
   ```

2. **Альтернативный источник** (Яндекс.Диск):
   ```
   https://disk.yandex.ru/d/ZxgB_kRu-mwqGg
   ```

### Установка ключа:

1. Загрузите файл `apt.gpg` на сервер
2. Скопируйте его в домашнюю директорию
3. Выполните команду:

```bash
sudo mv ~/apt.gpg /etc/apt/trusted.gpg.d/php.gpg
```

> **Примечание:** Ключ успешно установлен после выполнения команды.

## Проверка и обновление репозитория

1. Обновите список пакетов:

```bash
sudo apt update
```

2. Попробуйте установить PHP 8.3:

```bash
sudo apt install php8.3
```

## Устранение возможных проблем

Если возникают ошибки при выполнении `apt update`, проверьте:
- Правильность установки PGP-ключа
- Доступность репозиториев
- Наличие интернет-соединения

Для дополнительной проверки ключа выполните:
```bash
apt-key list
```

## 3. Установка Asterisk 21

### Загрузка и распаковка исходников
```bash
cd /usr/src
wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-21-current.tar.gz
tar xvf asterisk-21-current.tar.gz
cd asterisk-21*/
```

### Компиляция и установка
```bash
contrib/scripts/get_mp3_source.sh
contrib/scripts/install_prereq install
./configure --libdir=/usr/lib64 --with-pjproject-bundled --with-jansson-bundled
make menuselect
make
make install
make samples
make config
ldconfig
```

### Настройка пользователя Asterisk
```bash
groupadd asterisk
useradd -r -d /var/lib/asterisk -g asterisk asterisk
usermod -aG audio,dialout asterisk
chown -R asterisk:asterisk /etc/asterisk
chown -R asterisk:asterisk /var/{lib,log,spool}/asterisk
chown -R asterisk:asterisk /usr/lib64/asterisk
```

### Редактирование конфигурационных файлов
```bash
sed -i 's|#AST_USER|AST_USER|' /etc/default/asterisk
sed -i 's|#AST_GROUP|AST_GROUP|' /etc/default/asterisk
sed -i 's|;runuser|runuser|' /etc/asterisk/asterisk.conf
sed -i 's|;rungroup|rungroup|' /etc/asterisk/asterisk.conf
echo "/usr/lib64" >> /etc/ld.so.conf.d/x86_64-linux-gnu.conf
ldconfig
```

## 4. Настройка Apache
```bash
sed -i 's/\(^upload_max_filesize = \).*/\120M/' /etc/php/8.2/apache2/php.ini
sed -i 's/\(^memory_limit = \).*/\1256M/' /etc/php/8.2/apache2/php.ini
sed -i 's/^\(User\|Group\).*/\1 asterisk/' /etc/apache2/apache2.conf
sed -i 's/AllowOverride None/AllowOverride All/' /etc/apache2/apache2.conf
a2enmod rewrite
systemctl restart apache2
rm /var/www/html/index.html
```

## 5. Настройка ODBC
```bash
cat <<EOF > /etc/odbcinst.ini
[MySQL]
Description = ODBC for MySQL (MariaDB)
Driver = /usr/lib/x86_64-linux-gnu/odbc/libmaodbc.so
FileUsage = 1
EOF

cat <<EOF > /etc/odbc.ini
[MySQL-asteriskcdrdb]
Description = MySQL connection to 'asteriskcdrdb' database
Driver = MySQL
Server = localhost
Database = asteriskcdrdb
Port = 3306
Socket = /var/run/mysqld/mysqld.sock
Option = 3
EOF
```

## 6. Установка FreePBX
```bash
cd /usr/local/src
wget http://mirror.freepbx.org/modules/packages/freepbx/freepbx-17.0-latest-EDGE.tgz
tar zxvf freepbx-17.0-latest-EDGE.tgz
cd /usr/local/src/freepbx/
./start_asterisk start
./install -n
```

## 7. Установка модулей и завершение настройки
```bash
fwconsole ma installall
fwconsole reload
fwconsole restart
```

## 8. Настройка автозапуска
```bash
cat <<EOF > /etc/systemd/system/freepbx.service
[Unit]
Description=FreePBX VoIP Server
After=mariadb.service
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/fwconsole start -q
ExecStop=/usr/sbin/fwconsole stop -q
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable freepbx
```

## 9. Завершение установки

После выполнения всех команд система FreePBX будет установлена. Вы можете получить доступ к веб-интерфейсу по адресу:

```
http://ваш_IP_адрес/admin
```

> **Примечание:** Замените `ваш_IP_адрес` на реальный IP-адрес вашего сервера.
