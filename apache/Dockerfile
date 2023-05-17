##
# Base image
# Url https://github.com/docker-library/wordpress/blob/5f5172057791d1deb1919fdd1c5c45e7daae3362/latest/php8.0/apache/Dockerfile
##
FROM php:8.0-apache as base
# composer
COPY --from=composer:2 /usr/bin/composer /usr/bin/composer
# wp-cli
COPY --from=wordpress:cli /usr/local/bin/wp  /usr/local/bin/wp

# install the PHP extensions we need (https://make.wordpress.org/hosting/handbook/handbook/server-environment/#php-extensions)
RUN set -ex; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	\
	apt-get update; \
	apt-get install -y --no-install-recommends \
	ghostscript \
	libfreetype6-dev \
	libicu-dev \
	libjpeg-dev \
	libmagickwand-dev \
	libpng-dev \
	libwebp-dev \
	libzip-dev \
	; \
	\
	docker-php-ext-configure gd \
	--with-freetype \
	--with-jpeg \
	--with-webp \
	; \
	docker-php-ext-install -j "$(nproc)" \
	bcmath \
	exif \
	gd \
	intl \
	mysqli \
	zip \
	; \
	# https://pecl.php.net/package/imagick
	pecl install imagick-3.6.0; \
	docker-php-ext-enable imagick; \
	rm -r /tmp/pear; \
	\
	# some misbehaving extensions end up outputting to stdout 🙈 (https://github.com/docker-library/wordpress/issues/669#issuecomment-993945967)
	out="$(php -r 'exit(0);')"; \
	[ -z "$out" ]; \
	err="$(php -r 'exit(0);' 3>&1 1>&2 2>&3)"; \
	[ -z "$err" ]; \
	\
	extDir="$(php -r 'echo ini_get("extension_dir");')"; \
	[ -d "$extDir" ]; \
	# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
	apt-mark auto '.*' > /dev/null; \
	apt-mark manual $savedAptMark; \
	ldd "$extDir"/*.so \
	| awk '/=>/ { print $3 }' \
	| sort -u \
	| xargs -r dpkg-query -S \
	| cut -d: -f1 \
	| sort -u \
	| xargs -rt apt-mark manual; \
	\
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	apt-get install -y --no-install-recommends \
	zip \
	unzip \
	less \
    mariadb-client; \
	rm -rf /var/lib/apt/lists/*; \
	\
	! { ldd "$extDir"/*.so | grep 'not found'; }; \
	# check for output like "PHP Warning:  PHP Startup: Unable to load dynamic library 'foo' (tried: ...)
	err="$(php --version 3>&1 1>&2 2>&3)"; \
	[ -z "$err" ]

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN set -eux; \
	docker-php-ext-enable opcache; \
	{ \
	echo 'opcache.memory_consumption=128'; \
	echo 'opcache.interned_strings_buffer=8'; \
	echo 'opcache.max_accelerated_files=4000'; \
	echo 'opcache.revalidate_freq=2'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini
RUN { \
	# https://www.php.net/manual/en/errorfunc.constants.php
	# https://github.com/docker-library/wordpress/issues/420#issuecomment-517839670
	echo 'error_reporting = E_ERROR | E_WARNING | E_PARSE | E_CORE_ERROR | E_CORE_WARNING | E_COMPILE_ERROR | E_COMPILE_WARNING | E_RECOVERABLE_ERROR'; \
	echo 'display_errors = Off'; \
	echo 'display_startup_errors = Off'; \
	echo 'log_errors = On'; \
	echo 'error_log = /dev/stderr'; \
	echo 'log_errors_max_len = 1024'; \
	echo 'ignore_repeated_errors = On'; \
	echo 'ignore_repeated_source = Off'; \
	echo 'html_errors = Off'; \
	} > /usr/local/etc/php/conf.d/error-logging.ini

RUN set -eux; \
	a2enmod rewrite expires; \
	\
	# https://httpd.apache.org/docs/2.4/mod/mod_remoteip.html
	a2enmod remoteip; \
	{ \
	echo 'RemoteIPHeader X-Forwarded-For'; \
	# these IP ranges are reserved for "private" use and should thus *usually* be safe inside Docker
	echo 'RemoteIPTrustedProxy 10.0.0.0/8'; \
	echo 'RemoteIPTrustedProxy 172.16.0.0/12'; \
	echo 'RemoteIPTrustedProxy 192.168.0.0/16'; \
	echo 'RemoteIPTrustedProxy 169.254.0.0/16'; \
	echo 'RemoteIPTrustedProxy 127.0.0.0/8'; \
	} > /etc/apache2/conf-available/remoteip.conf; \
	a2enconf remoteip; \
	# https://github.com/docker-library/wordpress/issues/383#issuecomment-507886512
	# (replace all instances of "%h" with "%a" in LogFormat)
	find /etc/apache2 -type f -name '*.conf' -exec sed -ri 's/([[:space:]]*LogFormat[[:space:]]+"[^"]*)%h([^"]*")/\1%a\2/g' '{}' +; \
	mkdir /docker-entrypoint.d; \
	mkdir /var/bedrock && chown www-data:www-data /var/bedrock

COPY 000-default.conf /etc/apache2/sites-available/
COPY --chown=www-data --chmod=755 docker-entrypoint.sh /docker-entrypoint.sh
WORKDIR /var/bedrock
USER www-data

ENTRYPOINT [ "/docker-entrypoint.sh" ]
CMD ["apache2-foreground"]