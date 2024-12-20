ARG BUILD_FROM
# FROM $BUILD_FROM AS snapcast
FROM ghcr.io/hassio-addons/base:stable AS snapcast

RUN echo "@testing https://dl-cdn.alpinelinux.org/alpine/edge/testing" >> /etc/apk/repositories \
  && apk update
# Install snapcast-server and snapweb
RUN apk add --no-cache --upgrade snapcast-server \
  # Install latest release of snapweb
  # (its not shipped with snapcast-server, and not shipped with snapcast since 0.28.0 anymore)
  && wget -O /tmp/snapweb.zip https://github.com/badaix/snapweb/releases/latest/download/snapweb.zip \
  && unzip -o /tmp/snapweb.zip -d /usr/share/snapserver/snapweb/ \
  && rm /etc/init.d/snapcast-server \
  && rm /tmp/snapweb.zip

# Expose Ports
## Snapcast Ports: 1704-1705 1780
EXPOSE 1704-1705 1780

###
# Airplay with Shairport-Sync
# - install shairport-sync runtime dependencies
# - build shairport sync
FROM snapcast AS snapcast-airport
ARG AIRPLAY_VERSION
ENV BUILD_AIRPLAY_VERSION "${AIRPLAY_VERSION:-1}"

RUN apk add --no-cache dbus avahi shairport-sync

# RUN apk add --no-cache dbus popt openssl soxr avahi libconfig glib \
#   && apk add --no-cache --upgrade --virtual .build-deps git build-base autoconf automake libtool popt-dev openssl-dev soxr-dev avahi-dev libconfig-dev glib-dev \
#   && mkdir -p /app/build \
#   && CONF_OPTIONS="\
#       --with-stdout \
#       --with-dbus-interface \
#       --with-avahi \
#       --with-ssl=openssl \
#       --with-soxr \
#       --with-metadata" \
#   #
#   # Airplay-2 support: If enabled, add and build its dependencies
#   && if [ "${BUILD_AIRPLAY_VERSION}" -eq 2 ]; then \
#     CONF_OPTIONS="${CONF_OPTIONS} --with-airplay-2" \
#     && apk add --no-cache --upgrade ffmpeg libplist libsodium libgcrypt libuuid \
#     && apk add --no-cache --upgrade --virtual .build-deps-airplay2 ffmpeg-dev libplist-dev libsodium-dev libgcrypt-dev xxd \
#     && cd /app/build \
#     && git clone https://github.com/mikebrady/nqptp.git nqptp.git \
#     && cd nqptp.git \
#     && autoreconf -fi \
#     && ./configure \
#     && make \
#     && make install \
#   ; fi \
#   #
#   # Build shairport-sync with metadata, stdout and pipe support (apk repo is without)
#   && cd /app/build \
#   && git clone https://github.com/mikebrady/shairport-sync.git shairport-sync.git \
#   && cd shairport-sync.git \
#   && autoreconf -i -f \
#   && ./configure ${CONF_OPTIONS} \
#   && make \
#   && make install \
#   #
#   # Cleanup build environment
#   && apk del --purge .build-deps \
#   && if [ "${BUILD_AIRPLAY_VERSION}" -eq 2 ]; then apk del --purge .build-deps-airplay2; fi \
#   && rm -rf /app/build \
  #
  # Configure dbus-daemon and avahi-daemon for rootless execution
  # Ref: https://gnaneshkunal.github.io/avahi-docker-non-root.html
  RUN echo "<busconfig><listen>unix:path=/var/run/dbus/system_bus_socket</listen></busconfig>" > /usr/share/dbus-1/session.d/custom.conf \
  && mkdir -p /var/run/dbus \
  && chmod 777 /var/run/dbus/ \
  && chmod 777 /etc/avahi/avahi-daemon.conf \
  && mkdir -p /var/run/avahi-daemon \
  && chown avahi:avahi /var/run/avahi-daemon \
  && chmod 777 /var/run/avahi-daemon


###
# Build and install librespot (Spotify Client)
# - Disable all audio out plugins, as they are not needed.
FROM snapcast-airport AS snapcast-airport-spotify
RUN apk add --no-cache librespot@testing
# RUN mkdir -p /app/build \
#   && cd /app/build \
#   && apk add --no-cache --upgrade --virtual .build-deps git libconfig-dev cargo build-base cmake rust-bindgen clang18-dev \
#   && git clone -b master https://github.com/librespot-org/librespot.git librespot.git \
#   && cd librespot.git \
#   && cargo build --release --no-default-features --features with-avahi \
#   && cp ./target/release/librespot /usr/sbin/ \
#   && chmod +x /usr/sbin/librespot \
#   #
#   # Cleanup build environment
#   && cargo clean \
#   && apk del --purge .build-deps \
#   && rm -rf /app/build ~/.cargo


###
# Create final image
FROM snapcast-airport-spotify AS snapcast-extended

ENV TZ "Etc/UTC"

# Install and build steps
# - install shairport-sync runtime dependencies
# - build shairport sync
# - build librespot
# - install nginx
# RUN apk add --no-cache --upgrade tzdata curl \
#   && mkdir -p /app/config /app/data /app/certs/ \
#   #
#   # Install NGINX for SSL reverse proxy to webinterface
#   && apk add --no-cache --upgrade nginx \
#   && mkdir -p /run/nginx/  \
#   && chown -R snapcast:snapcast /var/lib/nginx /var/log /usr/lib/nginx /run/nginx /app/certs \
#   && chown -R snapcast:snapcast /app

COPY etc /etc
RUN chmod +x /etc/services.d/**/* \
&& chmod +x /etc/cont-init.d/*

# Add NGINX configuration
# ADD nginx.conf /etc/nginx/http.d/default.conf

# Copy setup and healtcheck script
# ADD setup /etc/cont-init.d/10-snapcast-server-config
# ADD finish /etc/services.d/snapserver/
# ADD --chown=snapcast:snapcast --chmod=0775 healthcheck.sh /app/healthcheck.sh

# USER snapcast:snapcast

VOLUME [ "/app/config", "/app/data", "/app/certs" ]

# HEALTHCHECK CMD [ "/bin/sh", "/app/healthcheck.sh" ]

# Expose Ports
## Snapcast Ports:   1704-1705 1780
## Shairport-Sync:
### Ref: https://github.com/mikebrady/shairport-sync/blob/master/TROUBLESHOOTING.md#ufw-firewall-blocking-ports-commonly-includes-raspberry-pi
### AirPlay ports:    3689/tcp 5000/tcp 6000-6009/udp
### AirPlay-2 ports:  3689/tcp 5000/tcp 6000-6009/udp 7000/tcp for airplay, 319-320/udp for NQPTP
### Avahi ports:      5353
## NGINX ports:      443 or user defined
EXPOSE 1704-1705 1780 4953 3689 5000 6000-6009/udp 7000 319-320/udp 5353 1788

ENTRYPOINT ["/init"]