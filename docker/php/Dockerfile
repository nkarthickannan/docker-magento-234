FROM php:7.2-fpm-buster

# Set working directory
WORKDIR /var/www/html

# Copy PHP configuration
COPY php.ini /usr/local/etc/php/conf.d/magento.ini

# Fix Debian Buster repositories (use archive) and update package lists
RUN echo 'deb http://archive.debian.org/debian buster main' > /etc/apt/sources.list && \
    echo 'deb http://archive.debian.org/debian-security buster/updates main' >> /etc/apt/sources.list && \
    apt-get update

# Install required system packages
RUN apt-get install -y \
    libzip-dev \
    libicu-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libfreetype6-dev \
    libxml2-dev \
    libxslt-dev \
    git

# Configure GD extension
RUN docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/

# Install PHP extensions
RUN docker-php-ext-install -j$(nproc) \
    gd \
    zip \
    intl \
    bcmath \
    pdo_mysql \
    xsl \
    soap \
    sockets

# Clean up apt cache
RUN rm -rf /var/lib/apt/lists/*

# Install Composer 1
RUN curl -sS https://getcomposer.org/installer | php -- --version=1.10.27 --install-dir=/usr/local/bin --filename=composer

# Set proper permissions for running as non-root user
RUN groupadd -g 1000 magento && \
    useradd -u 1000 -g magento -s /bin/bash magento && \
    chown -R magento:magento /var/www/html

# Use non-root user
USER magento

# Expose port for PHP-FPM
EXPOSE 9000

# Default command
CMD ["php-fpm"]
