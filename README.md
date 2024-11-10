# 一. 問題釐清

**Bookstack PDF轉出會亂碼的原因為:**

使用的container image的Alpine Linux 無安裝中文字體及Bookstack使用的PDF exporter不支援中文輸出格式，Bookstack官方使用的PDF Export 為 dompdf ，dompdf主要是用在Laravel 框架下的一項 HTML 內容轉換成 PDF 文件的工具，但由於dompdf處理中文亂碼的問題，過於複雜，且會增加重新打包image的流程，本次略過，找了網路上許多方法，都建議使用wkhtmltopdf來代替，但經過反覆測試都會失敗，後來查看官方文件，發現自從BookStack v24.05後，就不支援使用wkhtmltopdf，如下圖:
![image](https://github.com/steven502041/Bookstack-PDF-Export-/blob/main/img/wkhtmltopdf.png)

官方有給出別的替代方法，改使用Weasyprint，他也是一種把HTML轉換成PDF的工具，詳情可以瀏覽連結:

https://www.bookstackapp.com/docs/admin/pdf-rendering/

https://doc.courtbouillon.org/weasyprint/stable/

# 二. 解決辦法

 釐清了上述問題後，需要解決兩個問題

1. 需要重新打包image，加入免版權中文字檔並安裝weasyprint
2. 在部屬app時，需加入特定參數，讓他使用weasyprint，作為``EXPORT_PDF_COMMAND``的工具

下方解決流程，可搭配folder [docker-bookstack-master](https://github.com/steven502041/Bookstack-PDF-Export-/tree/main/docker-bookstack-master)，相互檢閱

### 一. 重新打包image

使用以下專案範例更改 : `solidnerd/bookstack:24.10.0`

1.1 去官方GitHub下載或git pull ，整個專案的repo

https://github.com/solidnerd/docker-bookstack

1.2 打開專案的dockerfile，增加安裝開源字體與weasyprint

本次安裝的字體是來自「 Google Noto Fonts 」所提供的字型[Noto Sans CJK TC](https://www.google.com/get/noto/#sans-hant)

```docker
FROM alpine:3 AS bookstack
ENV BOOKSTACK_VERSION=24.10.1
RUN apk add --no-cache curl tar
RUN set -x; \
    curl -SL -o bookstack.tar.gz https://github.com/BookStackApp/BookStack/archive/v${BOOKSTACK_VERSION}.tar.gz  \
    && mkdir -p /bookstack \
    && tar xvf bookstack.tar.gz -C /bookstack --strip-components=1 \
    && rm bookstack.tar.gz

FROM php:8.3-apache-bookworm AS final
RUN set -x; \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        git \
        zlib1g-dev \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng-dev  \
        libldap2-dev  \
        libtidy-dev  \
        libxml2-dev  \
        fontconfig  \
        fonts-freefont-ttf   \
        wget \
        tar \
        curl \
        libzip-dev \
        unzip \
        fonts-noto-cjk \      # 安裝開源字體Noto Sans CJK
        weasyprint \          # 安裝weasyprint
    && docker-php-ext-install -j$(nproc) dom pdo pdo_mysql zip tidy  \
    && docker-php-ext-configure ldap \
    && docker-php-ext-install -j$(nproc) ldap \
    && docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd
    
RUN a2enmod rewrite remoteip; \
    { \
    echo RemoteIPHeader X-Real-IP ; \
    echo RemoteIPTrustedProxy 10.0.0.0/8 ; \
    echo RemoteIPTrustedProxy 172.16.0.0/12 ; \
    echo RemoteIPTrustedProxy 192.168.0.0/16 ; \
    } > /etc/apache2/conf-available/remoteip.conf; \
    a2enconf remoteip

RUN set -ex; \
    sed -i "s/Listen 80/Listen 8080/" /etc/apache2/ports.conf; \
    sed -i "s/VirtualHost *:80/VirtualHost *:8080/" /etc/apache2/sites-available/*.conf

COPY bookstack.conf /etc/apache2/sites-available/000-default.conf

COPY --from=bookstack --chown=33:33 /bookstack/ /var/www/bookstack/

ARG COMPOSER_VERSION=2.7.6
RUN set -x; \
    cd /var/www/bookstack \
    && curl -sS https://getcomposer.org/installer | php -- --version=$COMPOSER_VERSION \
    && /var/www/bookstack/composer.phar install -v -d /var/www/bookstack/ \
    && rm -rf /var/www/bookstack/composer.phar /root/.composer \
    && chown -R www-data:www-data /var/www/bookstack 
COPY php.ini /usr/local/etc/php/php.ini
COPY docker-entrypoint.sh /bin/docker-entrypoint.sh

WORKDIR /var/www/bookstack

# www-data
USER 33

VOLUME ["/var/www/bookstack/public/uploads","/var/www/bookstack/storage/uploads"]

ENV RUN_APACHE_USER=www-data \
    RUN_APACHE_GROUP=www-data

EXPOSE 8080

ENTRYPOINT ["/bin/docker-entrypoint.sh"]

ARG BUILD_DATE
ARG VCS_REF
LABEL org.label-schema.build-date=$BUILD_DATE \
      org.label-schema.docker.dockerfile="/Dockerfile" \
      org.label-schema.license="MIT" \
      org.label-schema.name="bookstack" \
      org.label-schema.vendor="solidnerd" \
      org.label-schema.url="https://github.com/solidnerd/docker-bookstack/" \
      org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.vcs-url="https://github.com/solidnerd/docker-bookstack.git" \
      org.label-schema.vcs-type="Git"

```

由於上方官方提供的apt package，會包含安裝到Noto Sans CJK的多國語言系(包括JP、KR等等)，詳情可以參考下方連結

https://launchpad.net/ubuntu/jammy/+package/fonts-noto-cjk

為求image最小化，我是另外將繁中字體下載並納入使用 ( 使用者可以斟酌使用，容量差70MB)

![image](https://github.com/steven502041/Bookstack-PDF-Export-/blob/main/img/Noto%20Sans%20CJK.png)

```docker
FROM alpine:3 AS bookstack
ENV BOOKSTACK_VERSION=24.10.1
RUN apk add --no-cache curl tar
RUN set -x; \
    curl -SL -o bookstack.tar.gz https://github.com/BookStackApp/BookStack/archive/v${BOOKSTACK_VERSION}.tar.gz  \
    && mkdir -p /bookstack \
    && tar xvf bookstack.tar.gz -C /bookstack --strip-components=1 \
    && rm bookstack.tar.gz

FROM php:8.3-apache-bookworm AS final
RUN mkdir -p /usr/share/fonts/chinese         # 在container裡建立放入中文字體的目錄
COPY fonts/*.ttf /usr/share/fonts/chinese     # 在打包image的資料夾裡，新增fonts目錄，並把下載的字幕檔.tff放入裡面
RUN set -x; \
    apt-get update \
    && apt-get install -y --no-install-recommends \
        git \
        zlib1g-dev \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng-dev  \
        libldap2-dev  \
        libtidy-dev  \
        libxml2-dev  \
        fontconfig  \
        fonts-freefont-ttf   \
        wget \
        tar \
        curl \
        libzip-dev \
        unzip \
        weasyprint \
    && docker-php-ext-install -j$(nproc) dom pdo pdo_mysql zip tidy  \
    && docker-php-ext-configure ldap \
    && docker-php-ext-install -j$(nproc) ldap \
    && docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd
    
RUN a2enmod rewrite remoteip; \
    { \
    echo RemoteIPHeader X-Real-IP ; \
    echo RemoteIPTrustedProxy 10.0.0.0/8 ; \
    echo RemoteIPTrustedProxy 172.16.0.0/12 ; \
    echo RemoteIPTrustedProxy 192.168.0.0/16 ; \
    } > /etc/apache2/conf-available/remoteip.conf; \
    a2enconf remoteip

RUN set -ex; \
    sed -i "s/Listen 80/Listen 8080/" /etc/apache2/ports.conf; \
    sed -i "s/VirtualHost *:80/VirtualHost *:8080/" /etc/apache2/sites-available/*.conf

COPY bookstack.conf /etc/apache2/sites-available/000-default.conf

COPY --from=bookstack --chown=33:33 /bookstack/ /var/www/bookstack/

ARG COMPOSER_VERSION=2.7.6
RUN set -x; \
    cd /var/www/bookstack \
    && curl -sS https://getcomposer.org/installer | php -- --version=$COMPOSER_VERSION \
    && /var/www/bookstack/composer.phar install -v -d /var/www/bookstack/ \
    && rm -rf /var/www/bookstack/composer.phar /root/.composer \
    && chown -R www-data:www-data /var/www/bookstack 
COPY php.ini /usr/local/etc/php/php.ini
COPY docker-entrypoint.sh /bin/docker-entrypoint.sh

WORKDIR /var/www/bookstack

# www-data
USER 33

VOLUME ["/var/www/bookstack/public/uploads","/var/www/bookstack/storage/uploads"]

ENV RUN_APACHE_USER=www-data \
    RUN_APACHE_GROUP=www-data

EXPOSE 8080

ENTRYPOINT ["/bin/docker-entrypoint.sh"]

ARG BUILD_DATE
ARG VCS_REF
LABEL org.label-schema.build-date=$BUILD_DATE \
      org.label-schema.docker.dockerfile="/Dockerfile" \
      org.label-schema.license="MIT" \
      org.label-schema.name="bookstack" \
      org.label-schema.vendor="solidnerd" \
      org.label-schema.url="https://github.com/solidnerd/docker-bookstack/" \
      org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.vcs-url="https://github.com/solidnerd/docker-bookstack.git" \
      org.label-schema.vcs-type="Git"

```

1.3 修改dockerfile後，即可執行docker build指令

```docker
docker buildx build --platform linux/amd64 -t bookstack-chinese-pdf-exporter . 
```

### 二 . 加入特定參數，讓他使用weasyprint，作為`EXPORT_PDF_COMMAND`的工具

在啟動app時，加入特定參數參數，讓bookstack使用weasyprint作為PDF Exporter的工具，而並非原本的dompdf 

使用官方提供的docker-compse，將image換成剛剛打包好的image，或使用筆者已經打包好的image

下方提供docker-compse做為參考

```docker
version: '2'
services:
  mysql:
    image: mysql:8.3
    environment:
    - MYSQL_ROOT_PASSWORD= <your_MYSQL_ROOT_PASSWORD>
    - MYSQL_DATABASE=bookstack
    - MYSQL_USER=bookstack
    - MYSQL_PASSWORD=<your_MYSQL_PASSWORD>
    volumes:
    - mysql-data:/var/lib/mysql

  bookstack:
    image: steven502041/bookstack-chinese-pdf-exporter:24.10.1 # 在此使用剛剛打包的image
    depends_on:
    - mysql
    environment:
    - DB_HOST=mysql:3306
    - DB_DATABASE=bookstack
    - DB_USERNAME=bookstack
    - DB_PASSWORD=<your_MYSQL_PASSWORD>
    #set the APP_ to the URL of bookstack without without a trailing slash APP_URL=https://example.com
    - APP_URL=http://localhost:8080
    # APP_KEY is used for encryption where needed, so needs to be persisted to
    # preserve decryption abilities.
    # Can run `php artisan key:generate` to generate a key
    - APP_KEY=SomeRandomStringWith32Characters
    - EXPORT_PDF_COMMAND=weasyprint {input_html_path} {output_pdf_path}  # 加入此參數，讓bookstack使用weasyprint作為PDF Exporter的工具
    volumes:
    - uploads:/var/www/bookstack/public/uploads
    - storage-uploads:/var/www/bookstack/storage/uploads
    ports:
    - "1234:8080"

volumes:
 mysql-data:
 uploads:
 storage-uploads:
```