# matrix-synapse-federation
 *Тут ви можете знайти робочий проект для розгортання власного з двох наборів матричного сервера в docker-compose з автоматичним оновленням сертифікатів і робочим режимом федерації*

### **План кроків**

1. Встановити docker, docker-compose;
2. Створити дві головні директорії synapse, - для зберігання конфігів для бекенду і reverse_proxy, - для мережевої частини; 
3. Створити додаткові директорії: 
   - для бекенду: `./synapse/data_1`, `./synapse/data_2`, `./synapse/postgresql/postgresql_1`, `./synapse/postgresql/postgresql_2;`
     -  для мережі: ./reverse_proxy, ./reverse_proxy/synapse_federation;
4.  Створити файли: `docker-compose-synapse_v2.yml`,`docker-compose-reverse-proxy_v2.yml`, а також 2 файли у `/reverse_proxy/synapse_federation`, для увімкнення режиму федерації меж мікросервісами **synapse_1** і **synapse_2**.
     - 4.1. Згенерувати homeserver.yml для synapse_1 і synapse_2;
     - 4.2. Змінити БД з SQlite3 на PostgreSQL у homeserver.yml, або додати database.yml, а також інші точкові налаштування у директорію ./synapse/data_1/conf.d;
     - 4.3. Заповнити файли нашіх доменів у synapse_federation для .well-known/matrix/server і .well-known/matrix/client і вказати точки монтування у docker-compose-reverse-proxy_v2.yml;
 5. Після того як зробили підготовчі кроки, про всяк випадок перевірте все, перед запуском docker-compose-reverse-proxy_v2.yml, бо letsencrypt контейнер почне автоматичне створення сертифікатів.
 6. Можна перевірити конфіги через вбудувану утіліту -m synapse.config перевірки конфігів homeserver.yml.
 7. Спочатку запустіть компоуз із проксі docker-compose-reverse-proxy_v2.yml, щоб створилася мережа, а потім docker-compose-synapse_v2.yml. Якщо раптом не створилися сертифікати, то не вимикаючи компоуз із reverse proxy зупиніть і заново запустіть компоуз із бекендом synapse.

### Генерація homeserver.yml

Для початку нам потрібно згенерувати homeserver.yaml. Головна відмінність від встановлення через apt, що коли як пакет встановлюємо, то homeserver.yaml генерується автоматом у /etc/matrix-synapse/. А якщо вирішили встановлювати через docker, то потрібно спочатку згенерувати наш конфіг. Для цього нам доведеться створити папки synapse, - де будуть усі наші файли бекендів synapse_1 і synapse_2 зберігатися. Також створити у директорії synapse дві папки data_1 і data_2,  в які згенеруються файли homeserver.yml, директорії для ./postgresql/postgresql_1 і ./postgresql/postgresql_2.

Створимо директорію для збергіння конфігів homeserver.yml, а також postgresql:
```bash
su -l deploy
mkdir synapse && cd synapse | mkdir data_1 && mkdir data_2
deploy@new-matrix-demo:~$ ls -lArth synapse/docker-compose-synapse_v2.yml
-rw-r--r-- 1 deploy deploy 4.0K Jan 19 15:40 synapse/docker-compose-synapse_v2.yml
```
Збережемо файл у `/home/deploy/synapse/docker-compose-synapse_v2.yml`.
Генеруємо першим варіантом, якщо робили без компоуз, другим якщо через компоуз плануєте робити.
```bash
### Варіант №1. Потрібно буде виконати 2 рази, для stage-mchat.cosmonova-broadcast.tv і stage-matrix-chat.cosmonova-broadcast.tv
docker run -it --rm \
    -v /home/deploy/synapse/data_1:/data \
    -e SYNAPSE_SERVER_NAME=example-domain.name_one \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
Setting ownership on /data to 991:991
Creating log config /data/example-domain.name_one.log.config
Generating config file /data/homeserver.yaml
Generating signing key file /data/example-domain.name_one.signing.key
A config file has been generated in '/data/homeserver.yaml' for server name 'example-domain.name_one'. Please review this file and customise it to your needs.
### Варіант №2:
# Так само, спочатку згенеруємо для synapse_1. І потім запустимо ще раз, але з synapse_2
docker-compose -f docker-compose-synapse.yml run --rm synapse_1 generate
Creating synapse_postgresql_1 ... done
Creating synapse_synapse_1_run ... done
Setting ownership on /data to 991:991
Creating log config /data/example-domain.name_one.log.config
Generating config file /data/homeserver.yaml
Generating signing key file /data/example-domain.name_one.signing.key
A config file has been generated in '/data/homeserver.yaml' for server name 'example-domain.name_one'. Please review this file and customise it to your needs.
```
Якщо раптом після генерації власник не змінився, то змінемо власника вручну.
`sudo chown -R 991:991 /home/deploy/synapse/data_1 && sudo chown -R 991:991 /home/deploy/synapse/data_2`

### Базовий homeserver.yml

У вас повинен згенеруватися, ваш базовий конфіг, який ви потім будете корегувати під свої потреби. Те що було базове, я закоментував з коментарями, окрім усього іншого, згенерувалася останній рядок # vim:ft=yaml. Також на офіційному сайті [рекомендують](https://matrix-org.github.io/synapse/develop/setup/installation.html#debianubuntu) виносити додаткові налаштування, у тому числі і використання БД у окрему директорію conf.d.

> When installing with Debian packages, you might prefer to place files in /etc/matrix-synapse/conf.d/ to override your configuration without editing the main configuration file at /etc/matrix-synapse/homeserver.yaml. By doing that, you won't be asked if you want to replace your configuration file when you upgrade the Debian package to a later version.

_Конфіг БД, я вирішив у базовому прописати, бо не було часу тестувати чи спрацює таке розгалуження для контейнерів, для встановленого synapse через менеджер пакетів спраюцвало, а для контейнерів не тестував..._
```yaml
### /home/deploy/synapse/data_1/homeserver.yaml
# Configuration file for Synapse.
#
# This is a YAML file: see [1] for a quick introduction. Note in particular
# that *indentation is important*: all the elements of a list or dictionary
# should have the same indentation.
#
# [1] https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html
#
# For more information on how to configure Synapse, including a complete accounting of
# each option, go to docs/usage/configuration/config_documentation.md or
# https://element-hq.github.io/synapse/latest/usage/configuration/config_documentation.html
server_name: "example-domain.name_one"
pid_file: /data/homeserver.pid
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    resources:
      - names: [client, federation]
        compress: false
### This was generated by default
#database:
#  name: sqlite3
#  args:
#    database: /data/homeserver.db
###

database:
    name: psycopg2
    args:
        user: synapse
        password: synapse
        host: synapse_postgresql_1
        port: 5432
        database: synapse
        cp_min: 5
        cp_max: 10


log_config: "/data/example-domain.name_one.log.config"
media_store_path: /data/media_store
### secret keys will generate automatically
registration_shared_secret: "random_value_of_symbols_"
report_stats: true
macaroon_secret_key: "randmo_value_of_symbols"
form_secret: "random_value_of_symbols"
signing_key_path: "/data/example-domain.name_one.signing.key"
trusted_key_servers:
### This was generated by default "matrix.org"
  - server_name: "example-domain.name_one" #"matrix.org"
### this line generated to 
# vim:ft=yaml 
```

### Додавання federation і client

    Є багато способів додати підтримку федерації, але ми скористуємося найпростішим. Створимо файли, які потім примонтуємо у компоуз і nginx-proxy буде автоматично їх читати як один з параметрів nginx.conf. Створимо 2 файли конфігурації у директорії де знаходиться компоуз файл з reverse_proxy.

**URI** `/.well-known/matrix/server` використовується в режимі федерації для спрощення взаємодії між серверами. Коли інший сервер Matrix намагається зв'язатися з твоїм, чи іншим сервером (наприклад, при надсиланні повідомлень між користувачами різних серверів), він використовує цей URI для визначення того, як з'єднатися з твоїм сервером.

```bash
mkdir -p /home/deploy/reverse_proxy/synapse_federation
# в цій директорії створимо 2 файли example-domain.name_one і example-domain.name_two
### зміст файлу example-domain.name_one
# enable federation between different domains
location /.well-known/matrix/server {
    return 200 '{"m.server": "example-domain.name_one:443"}';
}

# It will allow users to enter their full username (e.g. @user:<server_name>) into clients which 
# supports well-known lookup to automatically configure the homeserver and identity server URLs. 
# This is useful so that users don't have to memorize or think about the actual homeserver URL you are using.
location /.well-known/matrix/client {
    return 200 '{"m.homeserver": {"base_url": "https://example-domain.name_one:8448"}}';
    default_type application/json;
    add_header Access-Control-Allow-Origin *;
}

### зміст файлу example-domain.name_two
# enable federation between different domains
location /.well-known/matrix/server {
    return 200 '{"m.server": "example-domain.name_two:443"}';
}

# It will allow users to enter their full username (e.g. @user:<server_name>) into clients which 
# supports well-known lookup to automatically configure the homeserver and identity server URLs. 
# This is useful so that users don't have to memorize or think about the actual homeserver URL you are using.    
location /.well-known/matrix/client {
    return 200 '{"m.homeserver": {"base_url": "https://example-domain.name_one:8448"}}';
    default_type application/json;
    add_header Access-Control-Allow-Origin *;
}
```

**URI** `/.well-known/matrix/client` використовується для спрощення процесу логіну користувачів у клієнти Matrix. Коли користувач вводить свій Matrix ID (наприклад, @admin:example-domain.name_one), клієнт може автоматично знайти правильний URL сервера. 

#### Відмінності налаштування режиму федерації через nginx та додатковій вказівці у homeserver.yml:

* `.well-known/matrix/server` є ключовим для функціонування федерації, особливо коли сервер стоїть за проксі або NAT.
* federation_whitelist є опціональним і використовується для більш специфічного управління федерацією.

Після того як ці файли зберегли, потрібно буде зв'язати їх зі шляхом у контейнері synapse_proxy. Зробимо ми це у докер-компоузі для мережі. Повний файл буде нижче, а зараз тільки приклад із оголошенням.

```yaml
version: "3.3"

services:
  synapse_proxy:
    image: "jwilder/nginx-proxy"
    container_name: "synapse_proxy"
    volumes:
### інші налаштування    
      - "/home/deploy/reverse_proxy/synapse_federation/example-domain.name_one:/etc/nginx/vhost.d/example-domain.name_one:ro"
      - "/home/deploy/reverse_proxy/synapse_federation/example-domain.name_two:/etc/nginx/vhost.d/example-domain.name_two:ro"
    networks:
### інші налаштування
```
Повна версія docker-compose у гілці dev.

### Перевірка конфіга homeserver.yml

Можна перевірити на наявність помилок наші конфіги.
```bash
docker exec -it synapse_2 python -m synapse.config -c /data/homeserver.yaml
Config parses OK!
docker exec -it synapse_1 python -m synapse.config -c /data/homeserver.yaml
Config parses OK!
```

### Реєстрація користувачів

На початку, потрібно буде створити користувачів через термінал. Це дозволить нам: по-перше, перевірити на практиці чи точно в нас усі мікросервіси правильно розгорнуті і налаштовані, ну і по-друге, дасть змогу заходити у webUI synapse-адмінки і далі усі маніпуляції проводити звідти
```bash
# Цим методом можна і адміньскі облікові записи робити. Також будьте уважні, якщо захочете створити для synapse_2, то незабудьте змінити домен на stage-matrix-chat
docker exec -it synapse_1 register_new_matrix_user https://example-domain.name_one -c /data/homeserver.yaml
New user localpart [root]: admin
Password: 
Confirm password: 
Make admin [no]: yes
Sending registration request...
Success!
```

#### Створення reverse-proxy

```bash
su -l deploy
mkdir reverse-proxy && cd !$ | mkdir data_1 && mkdir data_2
ls -lArth /home/deploy/reverse_proxy/docker-compose-reverse-proxy_v2.yml
-rw-r--r-- 1 deploy deploy 1,6K січ 24 10:04 /home/deploy/reverse_proxy/docker-compose-reverse-proxy_v2.yml
```

_Збережемо файл у `/home/deploy/reverse_proxy/docker-compose-reverse-proxy_v2.yml`_

```yaml
#### /home/deploy/reverse_proxy/docker-compose-reverse-proxy_v2.yml
version: "3.3"

services:
  synapse_proxy:
    image: "jwilder/nginx-proxy"
    container_name: "synapse_proxy"
    volumes:
      - "/etc/timezone:/etc/timezone:ro"
      - "/etc/localtime:/etc/localtime:ro"
      - "certs:/etc/nginx/certs"
      - "vhost:/etc/nginx/vhost.d:rw"
      - "html:/usr/share/nginx/html:rw"
      - "/run/docker.sock:/tmp/docker.sock:ro"
      - "/home/deploy/reverse_proxy/synapse_federation/example-domain.name_one:/etc/nginx/vhost.d/example-domain.name_one:ro"
      - "/home/deploy/reverse_proxy/synapse_federation/example-domain.name_two:/etc/nginx/vhost.d/example-domain.name_two:ro"
    networks:
      - synapse_network
    restart: "always"
    ports:
      - "80:80"
      - "443:443"

  synapse_letsencrypt:
    image: "jrcs/letsencrypt-nginx-proxy-companion"
    container_name: "letsencrypt"
    volumes:
      - "/etc/timezone:/etc/timezone:ro"
      - "/etc/localtime:/etc/localtime:ro"
      - "certs:/etc/nginx/certs:rw"
      - "vhost:/etc/nginx/vhost.d:rw"  
      - "html:/usr/share/nginx/html:rw"
      - "/run/docker.sock:/var/run/docker.sock:ro"
    environment:
      NGINX_PROXY_CONTAINER: "synapse_proxy"
### Added stage url for testing. To switch back, just comment out the ACME_CA_URI line
#      ACME_CA_URI: "https://acme-staging-v02.api.letsencrypt.org/directory"
    networks:
      - synapse_network
    restart: "always"
    depends_on:
      - "synapse_proxy"

networks:
  synapse_network:
    driver: bridge

volumes:
  certs:
  vhost:
  html:
root@server:~# cat !$
cat /home/deploy/reverse_proxy/docker-compose-reverse-proxy_v2.yml
version: "3.3"

services:
  synapse_proxy:
    image: "jwilder/nginx-proxy"
    container_name: "synapse_proxy"
    volumes:
      - "/etc/timezone:/etc/timezone:ro"
      - "/etc/localtime:/etc/localtime:ro"
      - "certs:/etc/nginx/certs"
      - "vhost:/etc/nginx/vhost.d:rw"
      - "html:/usr/share/nginx/html:rw"
      - "/run/docker.sock:/tmp/docker.sock:ro"
      - "/home/deploy/reverse_proxy/synapse_federation/example-domain.name_one:/etc/nginx/vhost.d/example-domain.name_one:ro"
      - "/home/deploy/reverse_proxy/synapse_federation/example-domain.name_two:/etc/nginx/vhost.d/example-domain.name_two:ro"
    networks:
      - synapse_network
    restart: "always"
    ports:
      - "80:80"
      - "443:443"

  synapse_letsencrypt:
    image: "jrcs/letsencrypt-nginx-proxy-companion"
    container_name: "letsencrypt"
    volumes:
      - "/etc/timezone:/etc/timezone:ro"
      - "/etc/localtime:/etc/localtime:ro"
      - "certs:/etc/nginx/certs:rw"
      - "vhost:/etc/nginx/vhost.d:rw"  
      - "html:/usr/share/nginx/html:rw"
      - "/run/docker.sock:/var/run/docker.sock:ro"
    environment:
      NGINX_PROXY_CONTAINER: "synapse_proxy"
### Added stage url for testing. To switch back, just comment out the ACME_CA_URI line
#      ACME_CA_URI: "https://acme-staging-v02.api.letsencrypt.org/directory"
    networks:
      - synapse_network
    restart: "always"
    depends_on:
      - "synapse_proxy"

# краще створити мережу з потрібною нам назвою, щоб докер кожного разу рандомною назвою не створював.
# Це дозволить уникнути будь-яких проблем з іншими контейнерами, які не використовують ту саму мережу.
# Використовуйте команду docker network create a network_name, щоб створити мережу
networks:
  synapse_network:
    driver: bridge

volumes:
  certs:
  vhost:
  html:
```

#### Запуск docker-compose-synapse_v2.yml

Після того як ви згенерували базові конфіги homeserver.yml та змінили БД на PostgreSQL, настав час запустити наші синапси наш synapse-compose для перевірки у боєвому режимі :-)
```bash
docker-compose -f synapse/docker-compose-synapse_v2.yml up -d
WARNING: Some networks were defined but are not used by any service: synapse_network
Creating network "synapse_default" with the default driver
Creating volume "synapse_postgresdata_1" with default driver
Creating volume "synapse_postgresdata_2" with default driver
### Піде збірка synapse файлів і адмінки
Building synapse-admin_1
[+] Building 345.4s (12/12) FINISHED 
 Image for service synapse-admin_1 was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Building synapse-admin_2
[+] Building 2.8s (12/12) FINISHED    
WARNING: Image for service synapse-admin_2 was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating synapse-admin_1      ... done
Creating synapse_postgresql_2 ... done
Creating synapse-admin_2      ... done
Creating synapse_postgresql_1 ... done
Creating synapse_2            ... done
Creating synapse_1            ... done

### Усього повинно бути запущеними 8 контейнерів: 2 від компоуз для letsencrypt і nginx-proxy, 6 від компоуз із бекендом, 2 бази даних, 2 адмінки і 2 сервіси synapse.
docker ps
CONTAINER ID   IMAGE                                    COMMAND                  CREATED         STATUS                   PORTS                                                                      NAMES
0f0684ae1a55   matrixdotorg/synapse:latest              "/start.py"              9 minutes ago   Up 9 minutes (healthy)   8009/tcp, 8448/tcp, 0.0.0.0:8009->8008/tcp, :::8009->8008/tcp              synapse_2
7cd941af53e8   matrixdotorg/synapse:latest              "/start.py"              9 minutes ago   Up 9 minutes (healthy)   8009/tcp, 0.0.0.0:8008->8008/tcp, :::8008->8008/tcp, 8448/tcp              synapse_1
7448649c26f6   postgres:latest                          "docker-entrypoint.s…"   9 minutes ago   Up 9 minutes             0.0.0.0:5433->5432/tcp, :::5433->5432/tcp                                  synapse_postgresql_2
59d911c87023   postgres:latest                          "docker-entrypoint.s…"   9 minutes ago   Up 9 minutes             0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                  synapse_postgresql_1
8c690cb7c1fc   synapse_synapse-admin_1                  "/docker-entrypoint.…"   9 minutes ago   Up 9 minutes             0.0.0.0:8080->80/tcp, :::8080->80/tcp                                      synapse-admin_1
7d83bc0eae3b   synapse_synapse-admin_2                  "/docker-entrypoint.…"   9 minutes ago   Up 9 minutes             0.0.0.0:8081->80/tcp, :::8081->80/tcp                                      synapse-admin_2
42799dd5eda2   jrcs/letsencrypt-nginx-proxy-companion   "/bin/bash /app/entr…"   3 days ago      Up 15 minutes                                                                                       letsencrypt
08869a21b085   jwilder/nginx-proxy                      "/app/docker-entrypo…"   3 days ago      Up 15 minutes            0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   synapse_proxy
```

### Запуск docker-compose-reverse-proxy

Зверніть увагу, що після першого запуску, створені контейнери автоматично полізуть до сайту let's encrypt реєструвати сертифікати, тому перед цим точно перевірите:
1) Чи всі домени ви правильно вказали;
2) Замініть сервер авторизації на стейджовий, там ліміт на видачу сертифікатів більший та і на сайті let's encrypt рекомендують для тестування використовувати тестове посилання.

Якщо щось буде не докінця налаштовано, або забули, наприклад додати режим федерації, то потім вас забанить і доведеться чекати, бо сертифікати не видадуться і витратите зайвий час на очікування.
```bash
docker-compose -f ./reverse_proxy/docker-compose-reverse-proxy_v2.yml up -d
Creating network "reverse_proxy_synapse_network" with driver "bridge"
Creating volume "reverse_proxy_certs" with default driver
Creating volume "reverse_proxy_vhost" with default driver
Creating volume "reverse_proxy_html" with default driver
Creating synapse_proxy ... done
Creating letsencrypt   ... done
# бачимо що все піднялося і працює
docker ps
CONTAINER ID   IMAGE                                    COMMAND                  CREATED          STATUS          PORTS                                                                                                                                                NAMES
cea563a593e2   jrcs/letsencrypt-nginx-proxy-companion   "/bin/bash /app/entr…"   49 seconds ago   Up 47 seconds                                                                                                                                                        letsencrypt
5b80c397f4e7   jwilder/nginx-proxy                      "/app/docker-entrypo…"   51 seconds ago   Up 48 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp, 0.0.0.0:81->80/tcp, :::81->80/tcp, 0.0.0.0:444->443/tcp, :::444->443/tcp   synapse_proxy
# подивимося логи letsencrypt
docker logs -f letsencrypt 
Info: running acme-companion version v2.2.10-5-g013005a
Warning: '/etc/acme.sh' does not appear to be a mounted volume.
Info: 4096 bits RFC7919 Diffie-Hellman group found, generation skipped.
Reloading nginx proxy (synapse_proxy)...
2024/01/24 10:17:35 Generated '/etc/nginx/conf.d/default.conf' from 8 containers
2024/01/24 10:17:35 [notice] 46#46: signal process started
2024/01/24 10:17:35 Generated '/app/letsencrypt_service_data' from 8 containers
2024/01/24 10:17:35 Running '/app/signal_le_service'
2024/01/24 10:17:35 Watching docker events
2024/01/24 10:17:35 Contents of /app/letsencrypt_service_data did not change. Skipping notification '/app/signal_le_service'
[Wed Jan 24 10:17:36 EET 2024] Create account key ok.
[Wed Jan 24 10:17:36 EET 2024] Registering account: https://acme-v02.api.letsencrypt.org/directory
[Wed Jan 24 10:17:38 EET 2024] Registered
[Wed Jan 24 10:17:38 EET 2024] ACCOUNT_THUMBPRINT='some_value'
Reloading nginx proxy (synapse_proxy)...
2024/01/24 10:17:38 Contents of /etc/nginx/conf.d/default.conf did not change. Skipping notification ''
2024/01/24 10:17:38 [notice] 59#59: signal process started
Reloading nginx proxy (synapse_proxy)...
2024/01/24 10:17:38 Contents of /etc/nginx/conf.d/default.conf did not change. Skipping notification ''
2024/01/24 10:17:38 [notice] 74#74: signal process started
Creating/renewal example-domain.name_one certificates... (example-domain.name_one example-admin-domain.name_one)
[Wed Jan 24 10:17:40 EET 2024] Using CA: https://acme-v02.api.letsencrypt.org/directory
[Wed Jan 24 10:17:40 EET 2024] Creating domain key
[Wed Jan 24 10:17:41 EET 2024] The domain key is here: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.key
[Wed Jan 24 10:17:41 EET 2024] Generate next pre-generate key.
[Wed Jan 24 10:17:42 EET 2024] Multi domain='DNS:example-domain.name_one,DNS:example-admin-domain.name_one'
[Wed Jan 24 10:17:42 EET 2024] Getting domain auth token for each domain
[Wed Jan 24 10:17:45 EET 2024] Getting webroot for domain='example-domain.name_one'
[Wed Jan 24 10:17:45 EET 2024] Getting webroot for domain='example-admin-domain.name_one'
[Wed Jan 24 10:17:46 EET 2024] Verifying: stage-matrix-chat.cosmonova-broadcast.tv
[Wed Jan 24 10:17:46 EET 2024] Pending, The CA is processing your order, please just wait. (1/30)
[Wed Jan 24 10:17:49 EET 2024] Success
[Wed Jan 24 10:17:49 EET 2024] Verifying: example-admin-domain.name_one
[Wed Jan 24 10:17:50 EET 2024] Pending, The CA is processing your order, please just wait. (1/30)
[Wed Jan 24 10:17:53 EET 2024] Success
[Wed Jan 24 10:17:53 EET 2024] Verify finished, start to sign.
[Wed Jan 24 10:17:53 EET 2024] Lets finalize the order.
[Wed Jan 24 10:17:53 EET 2024] Le_OrderFinalize='https://acme-v02.api.letsencrypt.org/acme/finalize/'
[Wed Jan 24 10:17:55 EET 2024] Downloading cert.
[Wed Jan 24 10:17:55 EET 2024] Le_LinkCert='https://acme-v02.api.letsencrypt.org/acme/cert/'
[Wed Jan 24 10:17:55 EET 2024] Cert success.
-----BEGIN CERTIFICATE-----
random_values_of_our_certificates
-----END CERTIFICATE-----
[Wed Jan 24 10:17:55 EET 2024] Your cert is in: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.cer
[Wed Jan 24 10:17:55 EET 2024] Your cert key is in: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.key
[Wed Jan 24 10:17:55 EET 2024] The intermediate CA cert is in: /etc/acme.sh/default/example-domain.name_one/ca.cer
[Wed Jan 24 10:17:55 EET 2024] And the full chain certs is there: /etc/acme.sh/default/example-domain.name_one/fullchain.cer
[Wed Jan 24 10:17:55 EET 2024] Your pre-generated next key for future cert key change is in: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.key.next
[Wed Jan 24 10:17:56 EET 2024] Installing cert to: /etc/nginx/certs/example-domain.name_one/cert.pem
[Wed Jan 24 10:17:56 EET 2024] Installing CA to: /etc/nginx/certs/example-domain.name_one/chain.pem
[Wed Jan 24 10:17:56 EET 2024] Installing key to: /etc/nginx/certs/example-domain.name_one/key.pem
[Wed Jan 24 10:17:56 EET 2024] Installing full chain to: /etc/nginx/certs/example-domain.name_one/fullchain.pem
Reloading nginx proxy (synapse_proxy)...
2024/01/24 10:17:56 Contents of /etc/nginx/conf.d/default.conf did not change. Skipping notification ''
2024/01/24 10:17:56 [notice] 89#89: signal process started
Creating/renewal example-domain.name_one certificates... (example-domain.name_one example-admin-domain.name_one)
[Wed Jan 24 10:17:57 EET 2024] Using CA: https://acme-v02.api.letsencrypt.org/directory
[Wed Jan 24 10:17:58 EET 2024] Creating domain key
[Wed Jan 24 10:17:58 EET 2024] The domain key is here: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.key
[Wed Jan 24 10:17:58 EET 2024] Generate next pre-generate key.
[Wed Jan 24 10:18:02 EET 2024] Multi domain='DNS:example-domain.name_one,DNS:example-admin-domain.name_one'
[Wed Jan 24 10:18:02 EET 2024] Getting domain auth token for each domain
[Wed Jan 24 10:18:05 EET 2024] Getting webroot for domain=''
[Wed Jan 24 10:18:05 EET 2024] Getting webroot for domain='example-domain.name_one'
[Wed Jan 24 10:18:05 EET 2024] Verifying: example-domain.name_one
[Wed Jan 24 10:18:06 EET 2024] Pending, The CA is processing your order, please just wait. (1/30)
[Wed Jan 24 10:18:09 EET 2024] Pending, The CA is processing your order, please just wait. (2/30)
[Wed Jan 24 10:18:12 EET 2024] Success
[Wed Jan 24 10:18:12 EET 2024] Verifying: example-admin-domain.name_one
[Wed Jan 24 10:18:13 EET 2024] Pending, The CA is processing your order, please just wait. (1/30)
[Wed Jan 24 10:18:15 EET 2024] Success
[Wed Jan 24 10:18:15 EET 2024] Verify finished, start to sign.
[Wed Jan 24 10:18:15 EET 2024] Lets finalize the order.
[Wed Jan 24 10:18:15 EET 2024] Le_OrderFinalize='https://acme-v02.api.letsencrypt.org/acme/finalize/'
[Wed Jan 24 10:18:17 EET 2024] Downloading cert.
[Wed Jan 24 10:18:17 EET 2024] Le_LinkCert='https://acme-v02.api.letsencrypt.org/acme/cert/'
[Wed Jan 24 10:18:17 EET 2024] Cert success.
-----BEGIN CERTIFICATE-----
random_values_of_our_certificates
-----END CERTIFICATE-----
[Wed Jan 24 10:18:17 EET 2024] Your cert is in: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.cer
[Wed Jan 24 10:18:17 EET 2024] Your cert key is in: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.key
[Wed Jan 24 10:18:17 EET 2024] The intermediate CA cert is in: /etc/acme.sh/default/example-domain.name_one/ca.cer
[Wed Jan 24 10:18:17 EET 2024] And the full chain certs is there: /etc/acme.sh/default/example-domain.name_one/fullchain.cer
[Wed Jan 24 10:18:17 EET 2024] Your pre-generated next key for future cert key change is in: /etc/acme.sh/default/example-domain.name_one/example-domain.name_one.key.next
[Wed Jan 24 10:18:18 EET 2024] Installing cert to: /etc/nginx/certs/example-domain.name_one/cert.pem
[Wed Jan 24 10:18:18 EET 2024] Installing CA to: /etc/nginx/certs/example-domain.name_one/chain.pem
[Wed Jan 24 10:18:18 EET 2024] Installing key to: /etc/nginx/certs/example-domain.name_one/key.pem
[Wed Jan 24 10:18:18 EET 2024] Installing full chain to: /etc/nginx/certs/example-domain.name_one/fullchain.pem
Reloading nginx proxy (synapse_proxy)...
2024/01/24 10:18:18 Contents of /etc/nginx/conf.d/default.conf did not change. Skipping notification ''
2024/01/24 10:18:18 [notice] 103#103: signal process started
Sleep for 3600s
```

Як бачимо із логів, після того як ми запустимо наші 2 компоуз файли reverse-proxy і synapse, то контейнери letsencrypt і nginx-proxy почнуть у автоматичному режимі створювати сертифікати для наших змінних оточення, які ми оголошували у docker-compose-synapse_v2.yml:

* VIRTUAL_HOST: "example-domain.name_one, example-admin-domain.name_one" у контейнері synapse_1;
* VIRTUAL_HOST: "example-domain.name_two, example-admin-domain.name_two" у контейнері synapse_2.

#### Мінуси методу з letsencrypt контейнером

На данний момент зіштовхнувся із наступними проблемами:

1. Висока складність підтримки контейнерів nginx і letsencrypt, через те що все відбувається під капотом. Конфіг nginx генерується при запуску контейнера і якщо volume примонтувати зі своїми конфігами, то jrcs/letsencrypt-nginx-proxy-companion не побачить їх;
2. Через автоматизацію неможливо власноруч ip-адреси прописувати;
3. Із попереднього витікає наступне, проблематично додати обмеження на доступ з IP-адрес до веб-амдінки;

Через автоматизацію отримання сертифікатів, можна отримати бан на 168 годин для зовнішьної IP-адреси. Наприклад, через криві руки, чи неуважність, чи якщо у вас автоматизовані конвейери котрі запускають деплой.

#### Баг чи фіча

> Після зупинки і запуску контейнерів доведеться власноруч прописати правильні IP-адреси для адмінок (тут мій косяк, трохи логіку у компоуз не так прописав). Потрібно буде підправити і винести домени для кожного сервісу окремо, а в мене в обох сервісах synapse_1 і synapse_2 ще додаткові домени для адмінок. Через це баг із ip-адресами і доводиться у контейнері прописувати вручну правилні ip-адреси ;


#### Додаткова інформація

Відносно необхідності змінювати порту для федерації, як пишуть у [офіційній документації](https://matrix-org.github.io/synapse/latest/delegate.html#delegation-faq) і ще [тут](https://matrix-org.github.io/synapse/develop/usage/configuration/config_documentation.html#serve_server_wellknown)...Скоріш за все у Matrix Synapse мають на увазі, якщо щось не працює на стандартному порту 8448, то потрібно це змінити:

1. **Стандартний Порт для Федерації:** За замовчуванням, Matrix Synapse використовує порт 8448 для федераційного трафіку (сервер-серверного зв'язку). Це стандартний порт, який використовується іншими серверами Matrix для спілкування з твоїм сервером.

2. **Зміна на Порт 443:** У деяких випадках, може виникнути необхідність змінити цей порт на 443, який є стандартним портом для HTTPS. Це може бути корисно у сценаріях, де порт 8448 заблокований або не доступний через мережеві обмеження.

3. **Файл .well-known/matrix/server:** Якщо плануєте змінити порт на 443, тобі потрібно буде налаштувати файл .well-known/matrix/server, щоб інші сервери Matrix могли визначити новий порт для федераційного з'єднання. Цей файл вказує на те, що сервер буде доступний на порту 443.

4. **Чи Потрібно Змінювати Порт?:** У нашому випадку, ми нічого не змінювали, бо все працює коректно і федерація налаштована правильно на порту 8448, то змінювати на порт 443 не обов'язково. Зміна порту зазвичай потрібна для випадків, коли існують специфічні обмеження або вимоги у твоєму мережевому середовищі.

### Корисні посилання

* [Installation Instructions](https://matrix-org.github.io/synapse/latest/setup/installation.html);
* [Registering a user in matryx-synapse](https://matrix-org.github.io/synapse/latest/setup/installation.html#registering-a-user);
* [Install certbot](https://www.atlantic.net/dedicated-server-hosting/how-to-install-matrix-synapse-with-nginx-and-lets-encrypt-ssl-on-debian-10/) for lets-encrypt support ;
* [Шаблон для nginx](https://matrix-org.github.io/synapse/latest/reverse_proxy.html#nginx) ;
* [Перевірка конфігу homeserver.yml](https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#config-validation);
* [Шаблон homeserver.yml](https://matrix-org.github.io/synapse/develop/usage/configuration/homeserver_sample_config.html#homeserver-sample-configuration-file);
