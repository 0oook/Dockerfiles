# syntax=docker/dockerfile:1

FROM --platform=$TARGETPLATFORM ubuntu:jammy as climber-build-stage
SHELL ["/bin/bash", "-eo", "pipefail", "-c"]
ARG DEBIAN_FRONTEND=noninteractive
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG EMAIL=my@example.com
ARG USERNAME=sea
ARG PASSWORD=123456
ARG DOMAIN=mydomain.com
ARG NEXTCLOUD_VERSION=28.0.1
ARG V2RAY_VERSION=v5.12.1
ARG CONFIG_DIR=/opt/config

# setup mirror sources
WORKDIR /root/ws
RUN <<EOF

apt update;
apt install -q -y --no-install-recommends \
    ca-certificates                       \
    tzdata                                \
    net-tools                             \
    locales                               \
    cron                                  \
    vim                                   \
    nginx                                 \
    zip                                   \
    unzip                                 \
    bzip2                                 \
    gnupg                                 \
    wget                                  \
    curl                                  \
    file                                  \
    git                                   \
    ssh                                   \
    iputils-ping                          \
    bash-completion


echo 'Asia/Shanghai' > /etc/timezone

ln -snf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

mkdir ${CONFIG_DIR}

# install acme
apt install -q -y --no-install-recommends socat
curl https://get.acme.sh | sh -s email=${EMAIL}

# generate SSL
#ls /root/.acme.sh/ -ahl
#/root/.acme.sh/acme.sh --upgrade
#/root/.acme.sh/acme.sh --issue -d ${DOMAIN} --standalone
#mkdir -p /etc/nginx/ssl/${EMAIL}
#/root/.acme.sh/acme.sh --install-cert -d ${DOMAIN} \
#--key-file       /etc/nginx/ssl/${EMAIL}/key.pem  \
#--fullchain-file /etc/nginx/ssl/${EMAIL}/cert.pem
#
#/root/.acme.sh/acme.sh --info -d ${DOMAIN}
#crontab  -l

# install Nextcloud
NEXTCLOUD_FILE_URL="https://download.nextcloud.com/server/releases/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2"
NEXTCLOUD_ASC_FILE_URL="https://download.nextcloud.com/server/releases/nextcloud-${NEXTCLOUD_VERSION}.tar.bz2.asc"
echo "Downloading nextcloud zip file, version ${NEXTCLOUD_VERSION}, from ${NEXTCLOUD_FILE_URL}"
curl -fSL -o nextcloud.tar.bz2      ${NEXTCLOUD_FILE_URL};
curl -fSL -o nextcloud.tar.bz2.asc  ${NEXTCLOUD_ASC_FILE_URL};
# gpg key from https://nextcloud.com/nextcloud.asc
gpg --batch --keyserver keyserver.ubuntu.com --recv-keys 28806A878AE423A28372792ED75899B9A724937A;
gpg --batch --verify nextcloud.tar.bz2.asc nextcloud.tar.bz2;
tar -xjvf nextcloud.tar.bz2 -C /var/www
chown -R www-data:www-data /var/www/nextcloud

apt install -q -y --no-install-recommends mariadb-server php-gd php-mysql php-curl php-mbstring php-intl php-gmp php-bcmath php-xml php-imagick php-zip php-fpm
/etc/init.d/mariadb start
cat << EOT > ./init.sql
CREATE USER '${USERNAME}'@'localhost' IDENTIFIED BY '${PASSWORD}';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO '${USERNAME}'@'localhost';
FLUSH PRIVILEGES;
EOT

cat << EOT > ./req.cnf
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no
[req_distinguished_name]
C = US
ST = VA
L = SomeCity
O = MyCompany
OU = MyDivision
CN = ${DOMAIN}
[v3_req]
keyUsage = critical, digitalSignature, keyAgreement
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = www.${DOMAIN}
DNS.2 = ${DOMAIN}
EOT
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ${CONFIG_DIR}/${DOMAIN}.key -out ${CONFIG_DIR}/${DOMAIN}.crt -config req.cnf -sha256
cat << EOT > /etc/nginx/sites-enabled/default
upstream php-handler {
    server unix:/var/run/php/php8.1-fpm.sock;
}

# Set the `immutable` cache control options only for assets with a cache busting `v` argument
map \$arg_v \$asset_immutable {
    "" "";
    default "immutable";
}


server {
    listen 80;
    listen [::]:80;
    server_name ${DOMAIN};

    # Prevent nginx HTTP Server Detection
    server_tokens off;

    # Enforce HTTPS
    return 301 https://\$server_name\$request_uri;
}

server {
    listen 443      ssl http2;
    server_name ${DOMAIN};

    # Path to the root of your installation
    root /var/www/nextcloud;

    # Use Mozilla's guidelines for SSL/TLS settings
    # https://mozilla.github.io/server-side-tls/ssl-config-generator/
    ssl_certificate     ${CONFIG_DIR}/${DOMAIN}.crt;
    ssl_certificate_key ${CONFIG_DIR}/${DOMAIN}.key;

    # Prevent nginx HTTP Server Detection
    server_tokens off;

    # HSTS settings
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    #add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload" always;

    # set max upload size and increase upload timeout:
    client_max_body_size 512M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml text/javascript application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Pagespeed is not supported by Nextcloud, so if your server is built
    # with the `ngx_pagespeed` module, uncomment this line to disable it.
    #pagespeed off;

    # The settings allows you to optimize the HTTP2 bandwitdth.
    # See https://blog.cloudflare.com/delivering-http-2-upload-speed-improvements/
    # for tunning hints
    client_body_buffer_size 512k;

    # HTTP response headers borrowed from Nextcloud `.htaccess`
    add_header Referrer-Policy                   "no-referrer"       always;
    add_header X-Content-Type-Options            "nosniff"           always;
    add_header X-Download-Options                "noopen"            always;
    add_header X-Frame-Options                   "SAMEORIGIN"        always;
    add_header X-Permitted-Cross-Domain-Policies "none"              always;
    add_header X-Robots-Tag                      "noindex, nofollow" always;
    add_header X-XSS-Protection                  "1; mode=block"     always;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Add .mjs as a file extension for javascript
    # Either include it in the default mime.types list
    # or include you can include that list explicitly and add the file extension
    # only for Nextcloud like below:
    include mime.types;
    types {
        text/javascript js mjs;
    }

    # Specify how to handle directories -- specifying `/index.php\$request_uri`
    # here as the fallback means that Nginx always exhibits the desired behaviour
    # when a client requests a path that corresponds to a directory that exists
    # on the server. In particular, if that directory contains an index.php file,
    # that file is correctly served; if it doesn't, then the request is passed to
    # the front-end controller. This consistent behaviour means that we don't need
    # to specify custom rules for certain paths (e.g. images and other assets,
    # `/updater`, `/ocm-provider`, `/ocs-provider`), and thus
    # `try_files \$uri \$uri/ /index.php\$request_uri`
    # always provides the desired behaviour.
    index index.php index.html /index.php\$request_uri;

    # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
    location = / {
        if ( \$http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/\$is_args\$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Make a regex exception for `/.well-known` so that clients can still
    # access it despite the existence of the regex rule
    # `location ~ /(\.|autotest|...)` which would otherwise handle requests
    # for `/.well-known`.
    location ^~ /.well-known {
        # The rules in this block are an adaptation of the rules
        # in `.htaccess` that concern `/.well-known`.

        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }

        location /.well-known/acme-challenge    { try_files \$uri \$uri/ =404; }
        location /.well-known/pki-validation    { try_files \$uri \$uri/ =404; }

        # Let Nextcloud's API for `/.well-known` URIs handle all other
        # requests by passing them to the front-end controller.
        return 301 /index.php\$request_uri;
    }

    # Rules borrowed from `.htaccess` to hide certain paths from clients
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:\$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

    # Ensure this block, which passes PHP files to the PHP process, is above the blocks
    # which handle static assets (as seen below). If this block is not declared first,
    # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
    # to the URI, resulting in a HTTP 500 error response.
    location ~ \.php(?:\$|/) {
        # Required for legacy support
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+|.+\/richdocumentscode\/proxy) /index.php\$request_uri;

        fastcgi_split_path_info ^(.+?\.php)(/.*)\$;
        set \$path_info \$fastcgi_path_info;

        try_files \$fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
        fastcgi_param PATH_INFO \$path_info;
        fastcgi_param HTTPS on;

        fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
        fastcgi_param front_controller_active true;     # Enable pretty urls
        fastcgi_pass php-handler;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;

        fastcgi_max_temp_file_size 0;
    }

    # Serve static files
    location ~ \.(?:css|js|mjs|svg|gif|png|jpg|ico|wasm|tflite|map)\$ {
        try_files \$uri /index.php\$request_uri;
        add_header Cache-Control "public, max-age=15778463, \$asset_immutable";
        access_log off;     # Optional: Don't log access to assets

        location ~ \.wasm\$ {
            default_type application/wasm;
        }
    }

    location ~ \.woff2?\$ {
        try_files \$uri /index.php\$request_uri;
        expires 7d;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    # Rule borrowed from `.htaccess`
    location /remote {
        return 301 /remote.php\$request_uri;
    }

    location / {
        try_files \$uri \$uri/ /index.php\$request_uri;
    }

    location /file {
        if (\$http_upgrade != "websocket") {
            return 404;
        }
        #proxy_redirect      on;
        proxy_pass          http://127.0.0.1:1234;
        proxy_http_version  1.1;
        proxy_set_header    Upgrade \$http_upgrade;
        proxy_set_header    Connection "upgrade";
        proxy_set_header    Host \$host;
        proxy_set_header    X-Real-IP \$remote_addr;
        proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto \$scheme;
    }

}
EOT

# install v2ray
if [ -z "${TARGETPLATFORM}" ]; then
  ARCH="64"
else
  case "${TARGETPLATFORM}" in
    linux/386)
      ARCH="32"
      ;;
    linux/amd64)
      ARCH="64"
      ;;
    linux/arm/v6)
      ARCH="arm32-v6"
      ;;
    linux/arm/v7)
      ARCH="arm32-v7a"
      ;;
    linux/arm64|linux/arm64/v8)
      ARCH="arm64-v8a"
      ;;
    *)
      ARCH=""
      ;;
  esac
fi
[ -z "${ARCH}" ] && echo "Error: Not supported OS Architecture" && exit 1

# Download files
V2RAY_FILE="v2ray-linux-${ARCH}.zip"
# https://github.com/v2fly/v2ray-core/releases/download/v5.12.1/v2ray-linux-64.zip
V2RAY_FILE_URL="https://github.com/v2fly/v2ray-core/releases/download/${V2RAY_VERSION}/${V2RAY_FILE}"
DGST_FILE="v2ray-linux-${ARCH}.zip.dgst"
# https://github.com/v2fly/v2ray-core/releases/download/v5.12.1/v2ray-linux-64.zip.dgst
DGST_FILE_URL="https://github.com/v2fly/v2ray-core/releases/download/${V2RAY_VERSION}/${DGST_FILE}"
echo "Downloading binary file: ${V2RAY_FILE}, version ${V2RAY_VERSION}, from ${V2RAY_FILE_URL}"
wget -O v2ray.zip ${V2RAY_FILE_URL}
echo "Downloading binary file: ${DGST_FILE}, version ${V2RAY_VERSION}, from ${DGST_FILE_URL}"
wget -O v2ray.zip.dgst ${DGST_FILE_URL}
if [ $? -ne 0 ]; then
    echo "Error: Failed to download binary file: ${V2RAY_FILE} ${DGST_FILE}" && exit 1
fi
echo "Download binary file: ${V2RAY_FILE} ${DGST_FILE} completed"

# Check SHA512
V2RAY_ZIP_HASH=$(sha512sum v2ray.zip | cut -f1 -d' ')
V2RAY_ZIP_DGST_HASH=$(cat v2ray.zip.dgst | grep -e 'SHA512' -e 'SHA2-512' | head -n1 | cut -f2 -d' ')
if [ "${V2RAY_ZIP_HASH}" = "${V2RAY_ZIP_DGST_HASH}" ]; then
    echo " Check passed"
else
    echo "V2RAY_ZIP_HASH: ${V2RAY_ZIP_HASH}"
    echo "V2RAY_ZIP_DGST_HASH: ${V2RAY_ZIP_DGST_HASH}"
    echo " Check have not passed yet " && exit 1
fi

# Prepare
echo "Prepare to use"
unzip v2ray.zip && chmod +x v2ray
mkdir -p /etc/v2ray /usr/local/share/v2ray /var/log/v2ray
mv v2ray /usr/bin/
mv geosite.dat geoip.dat /usr/local/share/v2ray/
mv config.json ${CONFIG_DIR}/config.json
ln -s ${CONFIG_DIR}/config.json /etc/v2ray/config.json

cat << EOT > /etc/init.d/v2ray
#!/bin/sh
### BEGIN INIT INFO
# Provides:          v2ray
# Required-Start:    \$network \$local_fs \$remote_fs
# Required-Stop:     \$remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: V2Ray proxy services
# Description:       V2Ray proxy services
### END INIT INFO

# Acknowledgements: Isulew Li <netcookies@gmail.com>

DESC=v2ray
NAME=v2ray
DAEMON=/usr/bin/v2ray
PIDFILE=/var/run/\$NAME.pid
SCRIPTNAME=/etc/init.d/\$NAME

DAEMON_OPTS="run -config /etc/v2ray/config.json"
TEST_OPTS="test -config /etc/v2ray/config.json"

# Exit if the package is not installed
[ -x \$DAEMON ] || exit 0

# Read configuration variable file if it is present
[ -r /etc/default/\$NAME ] && . /etc/default/\$NAME

# Load the VERBOSE setting and other rcS variables
. /lib/init/vars.sh

# Define LSB log_* functions.
# Depend on lsb-base (>= 3.0-6) to ensure that this file is present.
. /lib/lsb/init-functions

#
# Function that starts the daemon/service
#
do_start()
{
    # Take care of pidfile permissions
    mkdir /var/run/\$NAME 2>/dev/null || true

    # Return
    #   0 if daemon has been started
    #   1 if daemon was already running
    #   2 if daemon could not be started
    #   3 if configuration file not ready for daemon
    start-stop-daemon --start --quiet --pidfile \$PIDFILE --exec \$DAEMON -- \$TEST_OPTS || return 1
    start-stop-daemon --start --quiet --pidfile \$PIDFILE --exec \$DAEMON --background -m -- \$DAEMON_OPTS || return 2
}

#
# Function that stops the daemon/service
#
do_stop()
{
    # Return
    #   0 if daemon has been stopped
    #   1 if daemon was already stopped
    #   2 if daemon could not be stopped
    #   other if a failure occurred
    start-stop-daemon --stop --quiet --retry=TERM/30/KILL/5 --pidfile \$PIDFILE
    RETVAL="\$?"
    [ "\$RETVAL" = 2 ] && return 2
    # Wait for children to finish too if this is a daemon that forks
    # and if the daemon is only ever run from this initscript.
    # If the above conditions are not satisfied then add some other code
    # that waits for the process to drop all resources that could be
    # needed by services started subsequently.  A last resort is to
    # sleep for some time.
    start-stop-daemon --stop --quiet --oknodo --retry=0/30/KILL/5 --exec \$DAEMON
    [ "\$?" = 2 ] && return 2
    # Many daemons don't delete their pidfiles when they exit.
    rm -f \$PIDFILE
    return "\$RETVAL"
}

#
# Function that sends a SIGHUP to the daemon/service
#
do_reload() {
    #
    # If the daemon can reload its configuration without
    # restarting (for example, when it is sent a SIGHUP),
    # then implement that here.
    #
    start-stop-daemon --stop --signal 1 --quiet --pidfile \$PIDFILE
    return 0
}

case "\$1" in
  start)
    log_daemon_msg "Starting \$DESC " "\$NAME"
    do_start
    case "\$?" in
        0|1) log_end_msg 0 ;;
        2)   log_end_msg 1 ;;
    esac
  ;;
  stop)
    log_daemon_msg "Stopping \$DESC" "\$NAME"
    do_stop
    case "\$?" in
        0|1) log_end_msg 0 ;;
        2)   log_end_msg 1 ;;
    esac
    ;;
  status)
       status_of_proc "\$DAEMON" "\$NAME" && exit 0 || exit \$?
       ;;
  reload|force-reload)
    #
    # If do_reload() is not implemented then leave this commented out
    # and leave 'force-reload' as an alias for 'restart'.
    #
    log_daemon_msg "Reloading \$DESC" "\$NAME"
    do_reload
    log_end_msg \$?
    ;;
  restart|force-reload)
    log_daemon_msg "Restarting \$DESC" "\$NAME"
    do_stop
    case "\$?" in
      0|1)
        do_start
        case "\$?" in
            0) log_end_msg 0 ;;
            1) log_end_msg 1 ;; # Old process is still running
            *) log_end_msg 1 ;; # Failed to start
        esac
        ;;
      *)
        # Failed to stop
        log_end_msg 1
        ;;
    esac
    ;;
  *)
    echo "Usage: \$SCRIPTNAME {start|stop|status|reload|restart|force-reload}" >&2
    exit 3
    ;;
esac
EOT
chmod +x /etc/init.d/v2ray

cat << EOT > /startup.sh
#!/bin/bash
set -e
/etc/init.d/cron       start
/etc/init.d/mariadb    start
/etc/init.d/nginx      start
/etc/init.d/php8.1-fpm start
/etc/init.d/v2ray      start

while [ 1 ]; do
    date -R;
    sleep 1;
done
EOT

echo "" >> /root/.bashrc
echo "if [ -f /etc/bash_completion ] && ! shopt -oq posix; then" >> /root/.bashrc
echo "    . /etc/bash_completion" >> /root/.bashrc
echo "fi" >> /root/.bashrc
echo "" >> /root/.bashrc

EOF

# exec
ENTRYPOINT ["/bin/bash", "/startup.sh"]
# openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout private_key.pem -out public_certificate.crt
# https://github.com/v2fly/v2ray-core/releases/download/${TAG}/${V2RAY_FILE}
# https://github.com/v2fly/v2ray-core/releases/download/v5.12.1/v2ray-linux-64.zip
# mkdir -p /etc/v2ray /usr/local/share/v2ray /var/log/v2ray
# curl https://raw.githubusercontent.com/v2fly/docker/master/v2ray.sh | sh -s "" "v5.12.1"
