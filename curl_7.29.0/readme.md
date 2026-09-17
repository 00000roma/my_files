Сборка curl 7.29.0 с OpenSSL вместо NSS на CentOS 7.

Цель заменить системный curl 7.29.0, собранный с NSS, на curl той же версии, но собранный с OpenSSL. Сборка выполняется из исходников и устанавливается в отдельный каталог, чтобы не затрагивать системный /usr/bin/curl 

Исходные данные:
- ОС: CentOS 7 (x86_64)
- Целевая версия curl: 7.29.0

На момент начала работ CentOS 7 официально завершила жизненный цикл, поэтому перед установкой пакетов yum нужно переключиться на архивное зеркало.

### Шаг 1

Удалить старые конфигурации и создать новый файл репозитория:
```
# sudo rm -f /etc/yum.repos.d/*.repo
# nano /etc/yum.repos.d/CentOS-Vault.repo
```

```
[base]
name=CentOS-7.9.2009 - Base
baseurl=http://archive.kernel.org/centos-vault/7.9.2009/os/$basearch/
gpgcheck=0
enabled=1

[updates]
name=CentOS-7.9.2009 - Updates
baseurl=http://archive.kernel.org/centos-vault/7.9.2009/updates/x86_64/
gpgcheck=0
enabled=1

[extras]
name=CentOS-7.9.2009 - Extras
baseurl=http://archive.kernel.org/centos-vault/7.9.2009/extras/$basearch/
gpgcheck=0
enabled=1
```
Очистить кэш и проверить работоспособность:
```
# sudo yum clean all
# sudo yum makecache
```

### Шаг 2

Установить компилятор и библиотеки для сборки:
```
sudo yum install -y gcc make openssl-devel \
> libidn-devel libssh2-devel openldap-devel \
> krb5-devel c-ares-devel
```

Проверить установку:
```
gcc --version
rpm -q openssl-devel
```

### Шаг 3

Скачивание и распаковка исходников curl 7.29.0:
```
cd ~
wget --no-check-certificate https://curl.se/download/archeology/curl-7.29.0.tar.gz
```

Убеждаемся, что скачался именно архив:
```
file curl-7.29.0.tar.gz
```
ожидаемо gzip compressed data, а не HTML.

Распаковываем архив:
```
tar -xfz curl-7.29.0.tar.gz
cd curl-7.29.0
```

### Шаг 4

Конфигурация сборки:
```
./configure --prefix=/usr/local/curl-static-full \
            --without-nss \
            --with-ssl \
            --enable-ares \
            --with-gssapi \
            --with-libidn \
            --with-libssh2 \
            --enable-ldap --enable-ldaps \
            --disable-shared \
            --enable-static
```

| опция                                | назначение                                                         |
| ------------------------------------ | ------------------------------------------------------------------ |
| --prefix=/usr/local/curl-static-full | Установка в отдельный каталог, чтобы не затрагивать системный curl |
| --without-nss                        | Явный отказ от NSS                                                 |
| --with-ssl                           | Использовать OpenSSL                                               |
| --enable-ares                        | Асинхронный DNS (c-ares)                                           |
| --with-gssapi                        | Kerberos/GSS-аутентификация                                        |
| --with-libidn                        | Поддержка IDN                                                      |
| --with-libssh2                       | Поддержка SCP/SFTP                                                 |
| --enable-ldap --enable-ldaps         | Поддержка LDAP/LDAPS                                               |
| --disable-shared --enable-static     | Собрать libcurl статически (вшить в бинарник)                      |
Вывод успешной конфигурации: 

![Скриншот](<images/Pasted image 20260917111346.png>)

### Шаг 5

Сборка и установка:

```
make
sudo make install
```

Бинарник и сопутствующие файлы установятся в `/usr/local/curl-static-full/`:
- `/usr/local/curl-static-full/bin/curl`
- `/usr/local/curl-static-full/bin/curl-config`

### Шаг 6

Проверка.
Убеждаемся, что используется именно OpenSSL:
```
/usr/local/curl-static-full/bin/curl -V
```

ожидаемый вывод: 

![скриншот](<images/Pasted image 20260917121413.png>)

Финальная проверка https:

```
/usr/local/curl-static-full/bin/curl -v -I https://curl.se/ 2>&1 | grep -Ei "SSL connection|TLS"
```

![скриншот](<images/Pasted image 20260917113103.png>)


### Важные моменты

**Системный `/usr/bin/curl` не заменяется.** От него зависит `yum`, и его подмена привела бы к поломке пакетного менеджера. Собранный curl вызывается по полному пути или через симлинк/алиас под другим именем:

```
# sudo ln -s /usr/local/curl-static-full/bin/curl /usr/local/bin/curl2
# curl2 -V
```
![[Pasted image 20260917122004.png]]



**Ошибка `(60) SSL certificate problem`** при первом запуске решается установкой корневых сертификатов:

```
# sudo yum install -y ca-certificates
# sudo update-ca-trust force-enable
# sudo update-ca-trust extract
```


**Полная статическая сборка (`LDFLAGS="-static"`) на CentOS 7 невозможна** из-за отсутствия статических версий `libssl.a`, `libkrb5.a`, `libldap.a`. Используется компромиссный вариант: `libcurl` вшита, системные библиотеки — динамические.