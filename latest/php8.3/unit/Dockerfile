#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM unit:1.33.0-php8.3

# persistent dependencies
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
# Ghostscript is required for rendering PDF previews
		ghostscript \
	; \
	rm -rf /var/lib/apt/lists/*

# install the PHP extensions we need (https://make.wordpress.org/hosting/handbook/handbook/server-environment/#php-extensions)
RUN set -ex; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	\
	apt-get update; \
	apt-get install -y --no-install-recommends \
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
# https://github.com/Imagick/imagick/commit/5ae2ecf20a1157073bad0170106ad0cf74e01cb6 (causes a lot of build failures, but strangely only intermittent ones 🤔)
# see also https://github.com/Imagick/imagick/pull/641
# this is "pecl install imagick-3.7.0", but by hand so we can apply a small hack / part of the above commit
	curl -fL -o imagick.tgz 'https://pecl.php.net/get/imagick-3.7.0.tgz'; \
	echo '5a364354109029d224bcbb2e82e15b248be9b641227f45e63425c06531792d3e *imagick.tgz' | sha256sum -c -; \
	tar --extract --directory /tmp --file imagick.tgz imagick-3.7.0; \
	grep '^//#endif$' /tmp/imagick-3.7.0/Imagick.stub.php; \
	test "$(grep -c '^//#endif$' /tmp/imagick-3.7.0/Imagick.stub.php)" = '1'; \
	sed -i -e 's!^//#endif$!#endif!' /tmp/imagick-3.7.0/Imagick.stub.php; \
	grep '^//#endif$' /tmp/imagick-3.7.0/Imagick.stub.php && exit 1 || :; \
	docker-php-ext-install /tmp/imagick-3.7.0; \
	rm -rf imagick.tgz /tmp/imagick-3.7.0; \
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
		| awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); printf "*%s\n", so }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -rt apt-mark manual; \
	\
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
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
# https://wordpress.org/support/article/editing-wp-config-php/#configure-error-logging
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
# Upload size limits
RUN { \
		echo 'upload_max_filesize = 64M'; \
		echo 'post_max_size = 64M'; \
	} > /usr/local/etc/php/conf.d/upload-size.ini

RUN set -eux; \
	version='6.6.2'; \
	sha1='7acbf69d5fdaf804e3db322bad23b08d2e2e42ec'; \
	\
	curl -o wordpress.tar.gz -fL "https://wordpress.org/wordpress-$version.tar.gz"; \
	echo "$sha1 *wordpress.tar.gz" | sha1sum -c -; \
	\
# upstream tarballs include ./wordpress/ so this gives us /usr/src/wordpress
	tar -xzf wordpress.tar.gz -C /usr/src/; \
	rm wordpress.tar.gz; \
	\
# https://wordpress.org/support/article/htaccess/
	[ ! -e /usr/src/wordpress/.htaccess ]; \
	{ \
		echo '# BEGIN WordPress'; \
		echo ''; \
		echo 'RewriteEngine On'; \
		echo 'RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]'; \
		echo 'RewriteBase /'; \
		echo 'RewriteRule ^index\.php$ - [L]'; \
		echo 'RewriteCond %{REQUEST_FILENAME} !-f'; \
		echo 'RewriteCond %{REQUEST_FILENAME} !-d'; \
		echo 'RewriteRule . /index.php [L]'; \
		echo ''; \
		echo '# END WordPress'; \
	} > /usr/src/wordpress/.htaccess; \
	\
	chown -R unit:unit /usr/src/wordpress; \
# pre-create wp-content (and single-level children) for folks who want to bind-mount themes, etc so permissions are pre-created properly instead of root:root
# wp-content/cache: https://github.com/docker-library/wordpress/issues/534#issuecomment-705733507
	chown -R unit:unit /usr/src/wordpress/wp-content; \
	chmod -R 1777 /usr/src/wordpress/wp-content

VOLUME /var/www/html

COPY --chown=unit:unit wp-config-docker.php /usr/src/wordpress/

# NGINX Unit specific bits
WORKDIR /usr/src/wordpress
RUN set -eux; \
# save Unit's entrypoint so we can successfully/correctly configure Unit at runtime
	sed -e 's/docker-entrypoint/unit-entrypoint/g' /usr/local/bin/docker-entrypoint.sh > /usr/local/bin/unit-entrypoint.sh; \
	grep -F '/unit-entrypoint.d/' /usr/local/bin/unit-entrypoint.sh; \
	chmod +x /usr/local/bin/unit-entrypoint.sh; \
	\
	{ \
		echo; \
		echo '// WordPress does not support NGINX Unit (yet), so we have to do a little hackery to let it know that Unit supports nice permalinks (and prevent it from insisting on injecting "/index.php" in all the permalink options) -- the simplest way is teaching it to recognize Unit as if it were NGINX (which is only semi-true, but true enough for these purposes)'; \
		echo '$is_nginx = $is_nginx || str_starts_with($_SERVER["SERVER_SOFTWARE"], "Unit/");'; \
		echo '// see also https://github.com/WordPress/WordPress/blob/39f7f558d91afdd2f3afc7f3b049a6a800cd3f80/wp-includes/vars.php#L127'; \
	} >> /usr/src/wordpress/wp-config-docker.php
COPY unit.json /unit-entrypoint.d/wordpress.json

RUN cp /usr/src/wordpress/wp-config-docker.php /usr/src/wordpress/wp-config.php
COPY docker-entrypoint.sh /usr/local/bin/

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["unitd","--no-daemon","--control","unix:/var/run/control.unit.sock"]
