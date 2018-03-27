FROM alpine:3.7

LABEL maintainer="Maxime Bacoux <max@bacoux.com>"

ENV BIND_VERSION 9.12.1

RUN GPG_KEYS=BE0E9748B718253A28BB89FFF1B11BF05CF02E57 \
    && CONFIG="\
        --prefix=/usr \
        --sysconfdir=/etc/bind \
        --localstatedir=/var/bind \
        --enable-threads \
        --enable-largefile \
        --enable-ipv6 \
        --disable-linux-caps \
        --disable-backtrace \
        --disable-symtable \
        --with-pic \
        --with-gnu-ld \
    " \
    && apk add --no-cache --virtual .build-deps \
    gcc \
    libc-dev \
    make \
    openssl-dev \
    linux-headers \
    curl \
    gnupg \
    perl \
    perl-doc \
    && curl -fSL ftp://ftp.isc.org/isc/bind9/$BIND_VERSION/bind-$BIND_VERSION.tar.gz -o bind.tar.gz \
    && curl -fSL ftp://ftp.isc.org/isc/bind9/$BIND_VERSION/bind-$BIND_VERSION.tar.gz.asc -o bind.tar.gz.asc \
    && export GNUPGHOME="$(mktemp -d)" \
    && found=''; \
    for server in \
        ha.pool.sks-keyservers.net \
        hkp://keyserver.ubuntu.com:80 \
        hkp://p80.pool.sks-keyservers.net:80 \
        pgp.mit.edu \
    ; do \
        echo "Fetching GPG key $GPG_KEYS from $server"; \
        gpg --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$GPG_KEYS" && found=yes && break; \
    done; \
    test -z "$found" && echo >&2 "error: failed to fetch GPG key $GPG_KEYS" && exit 1; \
    gpg --batch --verify bind.tar.gz.asc bind.tar.gz; \
    pkill -9 gpg-agent; \
    rm -rf "$GNUPGHOME" bind.tar.gz.asc \
    && mkdir -p /usr/src \
    && tar -zxC /usr/src -f bind.tar.gz \
    && rm bind.tar.gz \
    && cd /usr/src/bind-$BIND_VERSION \
    && export STAGEDIR="$(mktemp -d)" \
    && ./configure $CONFIG \
    && make -j$(getconf _NPROCESSORS_ONLN) \
    && make DESTDIR=$STAGEDIR install \
    && install $STAGEDIR/usr/sbin/named /usr/sbin/named \
    && strip /usr/sbin/named \
    && rm -rf $STAGEDIR \
    && runDeps="$( \
        scanelf --needed --nobanner --format '%n#p' /usr/sbin/named \
            | tr ',' '\n' \
            | sort -u \
            | awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
    )" \
    && apk add --no-cache --virtual .bind-rundeps $runDeps \
    && apk del .build-deps \
    && mkdir -p /etc/bind \
    \
    && cd \
    && rm -rf /usr/src/bind-$BIND_VERSION

COPY named.conf /etc/bind/

EXPOSE 53/tcp 53/udp

CMD ["/usr/sbin/named", "-f"]
