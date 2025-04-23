# Установка FreePBX 17 на Debian 12

Подробная инструкция по установке FreePBX 17 на Debian 12.

## 1. Подготовка системы

### Вход в систему и обновление пакетов
```bash
sudo -i bash -c 'apt up && apt ug -y'
```

### Установка необходимых зависимостей
```bash
apt -y install build-essential git curl wget libnewt-dev libssl-dev libncurses5-dev subversion libsqlite3-dev libjansson-dev libxml2-dev uuid-dev default-libmysqlclient-dev htop sngrep lame ffmpeg mpg123 expect
```
### Удаление ненужных пакетов 
```bash 

```
## 2. Установка PHP 8.3

# Добавление нового PGP-ключа и обновление репозитория

## Добавление нового ключа PGP

```bash
apt install apt-transport-https
curl -sSLo /usr/share/keyrings/deb.sury.org-php.gpg https://packages.sury.org/php/apt.gpg
sh -c 'echo "deb [signed-by=/usr/share/keyrings/deb.sury.org-php.gpg] https://packages.sury.org/php/ $(lsb_release -sc) main" > /etc/apt/sources.list.d/php.list'
apt update
```
1. Обновите список пакетов:

```bash
apt update
```

2. Попробуйте установить PHP 8.3:

```bash
apt-get install -y linux-headers-`uname -r` openssh-server apache2 mariadb-server mariadb-client bison flex php8.3 php8.3-curl php8.3-cli php8.3-common php8.3-mysql php8.3-gd php8.3-mbstring php8.3-intl php8.3-xml php-pear sox pkg-config automake libtool autoconf unixodbc-dev uuid uuid-dev libasound2-dev libogg-dev libvorbis-dev libicu-dev libcurl4-openssl-dev odbc-mariadb libical-dev libneon27-dev libsrtp2-dev libspandsp-dev libtool-bin python-dev-is-python3 unixodbc software-properties-common nodejs npm ipset iptables fail2ban php8.3-soap
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
sed -i 's/\(^upload_max_filesize = \).*/\120M/' /etc/php/8.3/apache2/php.ini
sed -i 's/\(^memory_limit = \).*/\1256M/' /etc/php/8.3/apache2/php.ini
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
fwconsole ma disablerepo commercial
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
## 9. Удаление ненужных пакетов

```bash
apt-get remove -y build-essential git subversion bison flex pkg-config automake libtool autoconf libnewt-dev libncurses5-dev libsqlite3-dev libjansson-dev libxml2-dev uuid-dev default-libmysqlclient-dev python-dev-is-python3 unixodbc-dev htop sngrep expect && apt-get clean && apt-get autoremove -y && rm -rf /usr/src/asterisk-*/ && rm -f /usr/src/asterisk-*.tar.gz && rm -rf /var/lib/apt/lists/* && rm -rf /tmp/*
apt remove --purge -y apt-transport-https && apt autoremove -y && apt clean && rm -rf /var/lib/apt/lists/*
```
