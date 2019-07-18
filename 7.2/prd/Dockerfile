FROM php:7.2-fpm
MAINTAINER "Rus0 <andonid88@gmail.com>"

ENV PHP_EXTRA_CONFIGURE_ARGS="--enable-fpm --with-fpm-user=magento2 --with-fpm-group=magento2"

RUN echo "deb http://deb.debian.org/debian stretch-backports main" >> /etc/apt/sources.list

RUN apt-get update && apt-get install -my wget gnupg

RUN apt-key adv --keyserver pgp.mit.edu --recv-keys 5072E1F5 \
    && echo "deb http://repo.mysql.com/apt/debian/ jessie mysql-5.7" >> /etc/apt/sources.list

RUN apt-get update
RUN apt-get install -y \
    apt-utils \
    sudo \
    wget \
    unzip \
    cron \
    curl \
    libicu-dev \
    libxml2-dev libxslt1-dev \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libmcrypt4 \
    libcurl4-openssl-dev \
    git \
    vim \
    nano \
    openssh-server \
    supervisor \
    mysql-client \
    ocaml \
    expect \
    libsodium23

RUN curl -L https://github.com/bcpierce00/unison/archive/2.48.4.tar.gz | tar zxv -C /tmp && \
             cd /tmp/unison-2.48.4 && \
             sed -i -e 's/GLIBC_SUPPORT_INOTIFY 0/GLIBC_SUPPORT_INOTIFY 1/' src/fsmonitor/linux/inotify_stubs.c && \
             make && \
             cp src/unison src/unison-fsmonitor /usr/local/bin && \
             cd /root && rm -rf /tmp/unison-2.48.4 


RUN apt-get update && apt-get install -y pkg-config patch
ADD https://git.archlinux.org/svntogit/packages.git/plain/trunk/freetype.patch?h=packages/php /tmp/freetype.patch
RUN docker-php-source extract; \
    cd /usr/src/php; \
    patch -p1 -i /tmp/freetype.patch; \
    rm /tmp/freetype.patch

RUN docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-configure hash --with-mhash \
    && docker-php-ext-install -j$(nproc) intl xsl gd zip pdo_mysql pcntl opcache soap bcmath json iconv

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer 

RUN mkdir /var/run/sshd \
    && echo "StrictHostKeyChecking no" >> /etc/ssh/ssh_config

RUN apt-get update \
    && apt-get install -y apache2

RUN a2enmod rewrite \
    && a2enmod proxy \
    && a2enmod proxy_fcgi 

RUN rm -f /etc/apache2/sites-enabled/000-default.conf 

RUN useradd -m -d /home/magento2 -s /bin/bash magento2 && adduser magento2 sudo \
    && touch /etc/sudoers.d/privacy \
    && echo "Defaults        lecture = never" >> /etc/sudoers.d/privacy \
    && mkdir /home/magento2/magento2 && mkdir /home/magento2/state 

RUN curl -sS https://accounts.magento.cloud/cli/installer -o /home/magento2/installer \
    && rm -r /usr/local/etc/php-fpm.d/* \
    && sed -i 's/www-data/magento2/g' /etc/apache2/envvars

# Varnish
RUN apt-get update && apt-get install -y \
    automake \
    autotools-dev \
    libedit-dev \
    libjemalloc-dev \
    libncurses-dev \
    libpcre3-dev \
    libtool \
    pkg-config \
    python-docutils \
    python-sphinx \
    graphviz \
    gnupg


RUN apt-get update && apt-get install -y \
    software-properties-common \
    build-essential \
    zlib1g-dev \
    libncurses5-dev \
    libgdbm-dev \
    libnss3-dev \
    libssl-dev \
    libreadline-dev \
    libffi-dev \
    wget


RUN cd /tmp/ && curl -O https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tar.xz \
    && tar -xf Python-3.7.3.tar.xz \
    && cd Python-3.7.3 \
    && ./configure --enable-optimizations \
    && make \
    && make altinstall


# Add the Varnish Cache gpg key to the keyring used by APT:
RUN wget https://packagecloud.io/varnishcache/varnish5/gpgkey -O - | apt-key add - \
    && apt-get install apt-transport-https debian-archive-keyring -y \
    && echo "deb https://packagecloud.io/varnishcache/varnish5/debian/ stretch main" | tee -a /etc/apt/sources.list.d/varnishcache_varnish5.list \
    && echo "deb-src https://packagecloud.io/varnishcache/varnish5/debian/ stretch main" | tee -a /etc/apt/sources.list.d/varnishcache_varnish5.list

RUN apt-get update
RUN apt-get install varnish -y

RUN apt-get update && varnishd -V



RUN apt-get update && apt-get install -y \
    redis-server

RUN printf "\n \n" | pecl install redis && docker-php-ext-enable redis

#Varnish configurations



# PHP config
ADD conf/php.ini /usr/local/etc/php

# SSH config
COPY conf/sshd_config /etc/ssh/sshd_config
RUN chown magento2:magento2 /etc/ssh/ssh_config

# supervisord config
ADD conf/supervisord.conf /etc/supervisord.conf

# php-fpm config
ADD conf/php-fpm-magento2.conf /usr/local/etc/php-fpm.d/php-fpm-magento2.conf

# apache config
ADD conf/apache-default.conf /etc/apache2/sites-enabled/apache-default.conf

# unison script
ADD conf/.unison/magento2.prf /home/magento2/.unison/magento2.prf

ADD conf/unison.sh /usr/local/bin/unison.sh
ADD conf/entrypoint.sh /usr/local/bin/entrypoint.sh
ADD conf/check-unison.sh /usr/local/bin/check-unison.sh
RUN chmod +x /usr/local/bin/unison.sh && chmod +x /usr/local/bin/entrypoint.sh \
    && chmod +x /usr/local/bin/check-unison.sh

ENV USE_SHARED_WEBROOT 1
ENV SHARED_CODE_PATH /var/www/html
ENV WEBROOT_PATH /var/www/html
ENV MAGENTO_ENABLE_SYNC_MARKER 0


RUN chown -R magento2:magento2 /home/magento2 && \
    chown -R magento2:magento2 /var/www/html

# Delete user password to connect with ssh with empty password
RUN passwd magento2 -d

RUN cp /etc/varnish/default.vcl /etc/varnish/default.vcl.bk

ADD conf/default.vcl /etc/varnish/default.vcl

ADD conf/ports.conf /etc/apache2/ports.conf

ADD conf/000-default.conf /etc/apache2/sites-available/000-default.conf

ADD conf/varnish.conf /etc/default/varnish

# Elasticsearch

RUN wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -

RUN echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-7.x.list

RUN mkdir -p /usr/share/man/man1 && \
    apt-get update -y && \
    apt-get install -y openjdk-8-jdk

RUN apt-get install unzip -y && \
    apt-get autoremove -y

RUN apt-get update && apt-get install elasticsearch -y

COPY conf/elasticsearch/logging.yml /usr/share/elasticsearch/config/
COPY conf/elasticsearch/elasticsearch.yml /usr/share/elasticsearch/config/


EXPOSE 80 443 22 5000 44100
WORKDIR /var/www/html

USER root

ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
