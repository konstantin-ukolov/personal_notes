
Устанавливаем Java на ВМ

```bash
apt-get install java-11-openjdk -y
```

Выкачиваем нужную версию keycloak

```bash
wget https://github.com/keycloak/keycloak/releases/download/22.0.4/keycloak-22.0.4.tar.gz
```

разархивируем:

```bash
tar -xvzf keycloak-22.0.4.tar.gz
```

Переносим в директорию opt

```bash
mv keycloak-22.0.4 /opt/keycloak
```

Добавляем пользователя

```bash
groupadd keycloak
useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
```

Настраиваем права пользователю

```bash
chown -R keycloak: /opt/keycloak
chmod o+x /opt/keycloak/bin/
```

Keycloak service properties:
```bash
[Unit]

Description=Keycloak Open Source Identity and Access Management

After=syslog.target
After=network.target

[Service]

Type=simple
Restart=on-failure
RestartSec=2s

# Disable timeout logic and wait until process is stopped

TimeoutStopSec=0

# SIGTERM signal is used to stop the Java process

KillSignal=SIGTERM

# Send the signal only to the JVM rather than its control group

KillMode=process

# Java process is never killed

SendSIGKILL=no

# When a JVM receives a SIGTERM signal it exits with code 143

SuccessExitStatus=143
LimitMEMLOCK=infinity
LimitNOFILE=65535
User={{ kc_user }}
Group={{ kc_group | default(kc_user) }}
WorkingDirectory={{ kc_apphome }}

# Database

Environment=KC_DB={{ kc_db_type |default('mariadb') }}
Environment=KC_DB_URL_DATABASE={{ kc_db_name |default('keycloak')}}
Environment=KC_DB_PASSWORD={{ kc_db_pass}}
Environment=KC_DB_USERNAME={{ kc_db_user }}
Environment=KC_DB_URL_HOST={{ kc_db_host}}

# Proxy

Environment=KC_PROXY={{ kc_proxy_mode |default('reencrypt') }}
Environment=KC_HOSTNAME_STRICT={{ kc_strict_hostname |default('false') }}

# HTTPS

Environment=KC_HTTPS_PORT={{ kc_port }}

# Debian default ssl-cert snakeoil cert
Environment=KC_HTTPS_CERTIFICATE_FILE={{ kc_cert |default('/etc/ssl/certs/ssl-cert-snakeoil.pem') }}

Environment=KC_HTTPS_CERTIFICATE_KEY_FILE={{ kc_cert_key |default('/etc/ssl/private/ssl-cert-snakeoil.key') }}

{% if kc_admin is defined and kc_admin_pass is defined %}

# Default user

Environment=KEYCLOAK_ADMIN={{ kc_admin }}
Environment=KEYCLOAK_ADMIN_PASSWORD={{ kc_admin_pass }}

{% endif %}

{% if kc_port |int <= 1024 %}

# Allow ports below 1024.

CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE

{% endif %}

ExecStart={{ kc_apphome }}/bin/kc.sh --verbose start --spi-connections-jpa-quarkus-migration-strategy=update --auto-build

[Install]

WantedBy=multi-user.target
```

C ПРОДА:

```bash
[Unit]

Description=Keycloak
After=network.target

[Service]

User=keycloak
Group=keycloak
LimitNOFILE=102642

EnvironmentFile=/opt/keycloak/conf/keycloak.conf

ExecStart=/opt/keycloak/bin/kc.sh start --hostname=prod-keycloak1.ecp.prod --hostname-strict-https=false --https-certificate-file=/opt/keycloak/conf/tls.crt --https-certificate-key-file=/opt/keycloak/conf/tls_key.key --optimized

[Install]

WantedBy=multi-user.target
```

#keycloak #auth