# Netbox via Podman

In diesem Projekt sind die Podman Quadlet Definitionen für Netbox festgehalten. Dadurch werden die Container, die Volumes und das Netzwerk erzeugt.
Secrets und Konfiguration müssen seperat erzeugt werden.

Die Container basieren auf dem [netbox-docker](https://github.com/netbox-community/netbox-docker) Projekt.
Dort giobt es auch weitere Informationen in der [Wiki](https://github.com/netbox-community/netbox-docker/wiki/)

## Installation

Die Dateien in `containers`, `images`, `networks` und `volumes` nach `/etc/containers/systemd/` kopieren. Die Ordnerstruktur kann übernommen werden, alle Dateien unterhalb von systemd werden eingelesen.

```shell
[root@podman ~]# tree /etc/containers/systemd/
/etc/containers/systemd/
└── netbox
    ├── containers
    │   ├── netbox.container
    │   ├── netbox-housekeeping.container
    │   ├── netbox-postgres.container
    │   ├── netbox-redis-cache.container
    │   ├── netbox-redis.container
    │   └── netbox-worker.container
    ├── images
    │   ├── netbox.image
    │   ├── netbox-postgres.image
    │   └── netbox-valkey.image
    ├── networks
    │   └── netbox.network
    └── volumes
        ├── netbox-media-files.volume
        ├── netbox-postgres-data.volume
        ├── netbox-redis-cache-data.volume
        ├── netbox-redis-data.volume
        ├── netbox-reports-files.volume
        └── netbox-scripts-files.volume
```

### Konfiguration

Die Konfiguration wird in `/opt/netbox/netbox.env` vorgenommen. Zudem benötigt man den `configuration` Ordner aus dem Git Projekt

```shell
git clone https://github.com/netbox-community/netbox-docker.git /tmp/netbox-docker
mkdir -p /opt/netbox/
cp -au /tmp/netbox-docker/configuration /opt/netbox/
cat << EOF > /opt/netbox/netbox.env
CORS_ORIGIN_ALLOW_ALL=True
DB_HOST=netbox-postgres
DB_NAME=netbox
DB_USER=netbox
EMAIL_FROM=netbox@bar.com
EMAIL_PASSWORD=
EMAIL_PORT=25
EMAIL_SERVER=localhost
EMAIL_SSL_CERTFILE=
EMAIL_SSL_KEYFILE=
EMAIL_TIMEOUT=5
EMAIL_USERNAME=netbox
# EMAIL_USE_SSL and EMAIL_USE_TLS are mutually exclusive, i.e. they can't both be `true`!
EMAIL_USE_SSL=false
EMAIL_USE_TLS=false
GRAPHQL_ENABLED=true
HOUSEKEEPING_INTERVAL=86400
MEDIA_ROOT=/opt/netbox/netbox/media
METRICS_ENABLED=false
REDIS_CACHE_DATABASE=1
REDIS_CACHE_HOST=netbox-redis-cache
REDIS_CACHE_INSECURE_SKIP_TLS_VERIFY=false
REDIS_CACHE_SSL=false
REDIS_DATABASE=0
REDIS_HOST=netbox-redis
REDIS_INSECURE_SKIP_TLS_VERIFY=false
REDIS_SSL=false
RELEASE_CHECK_URL=https://api.github.com/repos/netbox-community/netbox/releases
SKIP_SUPERUSER=true
WEBHOOKS_ENABLED=true
EOF
```

### Secrets

Kennwörter für Redis, die DB und das Netbox Secret werden via Podman Secrets eingelesen. Diese müssen mit `podman secret create` erstellt werden.

`netbox_auth_ldap_bind_password` und `netbox_superuser_password` sind Optional, bzw abhängig von der gewählten Konfiguration.

```shell
printf $(pwgen 32 1) | podman secret create netbox_redis_password -
printf $(pwgen 32 1) | podman secret create netbox_redis_cache_password -
printf $(pwgen 32 1) | podman secret create netbox_db_password -
printf $(pwgen 32 1) | podman secret create netbox_superuser_password -
printf $(pwgen 32 1) | podman secret create netbox_auth_ldap_bind_password -
printf $(pwgen -ys 64 1) | podman secret create netbox_secret_key -
```

## Start von Netbox

Nach dem Kopieren der Quadlets ein `systemctl daemon-reload` ausführen. Dann können die Objekte gestartet werden.

```shell
systemctl start netbox
```

Netbox ist daraufhin unter `localhost:8000` erreichbar. Ein Reverse-Proxy sollte eingerichtet werden.

### ReverseProxy via Caddy

[Caddy](https://caddyserver.com/) eignet sich sehr gut als Reverse Proxy, da er einfach zu konfigurieren ist und automatisch Zertifikate erstellt. Caddy kann aus EPEL installiert werden.

```shell
dnf install -y caddy
cat << EOF > /etc/caddy/Caddyfile.d/netbox.caddyfile
netbox.example.com {
  tls internal
  reverse_proxy :8000
}
EOF
systemctl enable --now caddy.service
```

Dadurch wird ein Self-Signed Zertifikat erstellt und verwendet.

Möchte man ein bestehendes Zertifikat nutzen, wäre die Syntax:

```shell
netbox.example.com {
  tls /etc/pki/tls/certs/cert.pem /etc/pki/tls/private/privkey.pem
  reverse_proxy :8000
}
```

Fun Fact:
Ist die VM öffentlich erreichbar, und der DNS stimmt, dann wird automatisch ein LetsEncrypt Zertifikat angefordert.

```shell
netbox.etes.de {
  reverse_proxy :8000
}
```

## Updates

Die Container haben alle die Option `AutoUpdate=registry` gesetzt, dadurch können die Images via `podman auto-update` aktualisiert werden.

Major Versionen werden dabei nicht berücksichtigt, da die Image Tags nicht auf `latest` stehen, sondern ein Tag mit Version haben.

Möchte man Netbox z.B. von 4.0 auf 4.1 aktualisieren, muss man in den Definitionen des Images (`/etc/containers/systemd/netbox/images/netbox.image`) das Tag anpassen.

`Image=docker.io/netboxcommunity/netbox:v4.0` -> `Image=docker.io/netboxcommunity/netbox:v4.1`

Danach die Container neustarten

```shell
systemctl restart netbox
```

## Troubleshooting

### Logging für ldap aktivieren

Möchte man LDAP einrichten und läuft in Fehler, weiß man oft nicht, warum. Die Netbox Container sind sehr spärlich mit logging. Das kann man in `/opt/netbox/configuration/logging.py` anpassen.
Zusätzlich setzt man in `/opt/netbox/netbox.env` noch die Variable `LOGLEVEL=DEBUG`

```python
# Remove first comment(#) on each line to implement this working logging example.
# Add LOGLEVEL environment variable to netbox if you use this example & want a different log level.
from os import environ

# Set LOGLEVEL in netbox.env or docker-compose.overide.yml to override a logging level of INFO.
LOGLEVEL = environ.get('LOGLEVEL', 'INFO')

LOGGING = {
   'version': 1,
   'disable_existing_loggers': False,
   'formatters': {
       'verbose': {
           'format': '{levelname} {asctime} {module} {process:d} {thread:d} {message}',
           'style': '{',
       },
       'simple': {
           'format': '{levelname} {message}',
           'style': '{',
       },
   },
   'filters': {
       'require_debug_false': {
           '()': 'django.utils.log.RequireDebugFalse',
       },
   },
   'handlers': {
       'console': {
           'level': LOGLEVEL,
           'filters': ['require_debug_false'],
           'class': 'logging.StreamHandler',
           'formatter': 'simple'
       },
       'mail_admins': {
           'level': 'ERROR',
           'class': 'django.utils.log.AdminEmailHandler',
           'filters': ['require_debug_false']
       }
   },
   'loggers': {
       'django': {
           'handlers': ['console'],
           'propagate': True,
       },
       'django.request': {
           'handlers': ['mail_admins'],
           'level': 'ERROR',
           'propagate': False,
       },
       'django_auth_ldap': {
           'handlers': ['console',],
           'propagate': True,
           'level': LOGLEVEL,
       }
   }
}
```
