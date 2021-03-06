FROM debian:9.11-slim

# util
RUN apt update \
    && apt install -y ca-certificates apt-transport-https gnupg2 curl lsb-release git unzip

# install php-fpm
RUN curl -fsSL https://packages.sury.org/php/apt.gpg | apt-key add \
    && echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/php7.list \
    && apt update -y \
    && apt upgrade -y \
    && mkdir -p /run/php/ \
    && apt install -y php7.3-fpm

# setup php
RUN mkdir -p /var/www/html \
    && sed -i 's/\/run\/php\/php7.3-fpm.sock/9000/' /etc/php/7.3/fpm/pool.d/www.conf \
    && sed -i 's/;catch_workers_output = yes/catch_workers_output = yes\ndecorate_workers_output = no/' /etc/php/7.3/fpm/pool.d/www.conf \
    && sed -i 's/error_log = \/var\/log\/php7.3-fpm.log/error_log = \/proc\/self\/fd\/2/' /etc/php/7.3/fpm/php-fpm.conf \
    && sed -i 's/;daemonize = yes/daemonize = no/' /etc/php/7.3/fpm/php-fpm.conf

# mssql driver
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
    && curl https://packages.microsoft.com/config/debian/9/prod.list > /etc/apt/sources.list.d/mssql-release.list \
    && apt update \
    && ACCEPT_EULA=Y apt install -y msodbcsql17 mssql-tools \
    && export PATH="$PATH:/opt/mssql-tools/bin" \
    && apt install -y unixodbc-dev

# install php ext
RUN apt install -y php7.3-dev php7.3-xml php7.3-bcmath php7.3-mbstring php7.3-mysql php7.3-zip locales \
    && sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen \
    && locale-gen \
    && pecl install sqlsrv pdo_sqlsrv \
    && printf "; priority=20\nextension=sqlsrv.so\n" > /etc/php/7.3/mods-available/sqlsrv.ini \
    && printf "; priority=30\nextension=pdo_sqlsrv.so\n" > /etc/php/7.3/mods-available/pdo_sqlsrv.ini \
    && phpenmod -v 7.3 sqlsrv pdo_sqlsrv

# composer
RUN curl -s https://getcomposer.org/installer | php \
    && mv composer.phar /usr/local/bin/composer

# cron
RUN apt install -y cron \
    && (crontab -l; echo "* * * * * cd /var/www/html && php artisan schedule:run >> /dev/null 2>&1") | crontab -

# supervisor
COPY ./docker/prod/php/laravel.conf /etc/supervisor/conf.d/laravel.conf
RUN apt install -y supervisor \
    && sed -i 's/\[supervisord\]/[supervisord]\nnodaemon = true/g' /etc/supervisor/supervisord.conf

# source
COPY . /var/www/html

# build
WORKDIR /var/www/html
RUN chown -R www-data:www-data /var/www/html \
    && php -r "file_exists('.env') || copy('.env.example', '.env');" \
    && composer install \
    && php artisan key:generate

EXPOSE 9000

CMD ["/usr/bin/supervisord","-c","/etc/supervisor/supervisord.conf"]
