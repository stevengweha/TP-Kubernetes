# Utiliser une image PHP officielle avec Apache
FROM php:8.1-apache

# Installer les extensions PHP nécessaires
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    git \
    curl \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd

# Installer Composer
COPY --from=composer:2.6 /usr/bin/composer /usr/bin/composer

# Copier les fichiers de l'application
COPY . /var/www/html

# Configurer les permissions
RUN chown -R www-data:www-data /var/www/html

# Activer le module Apache mod_rewrite
RUN a2enmod rewrite

# Exposer le port 80
EXPOSE 80

# Commande par défaut
CMD ["apache2-foreground"]