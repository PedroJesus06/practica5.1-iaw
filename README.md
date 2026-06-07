# Despliegue de PrestaShop Securizado con Docker Compose y HTTPS-Portal

El objetivo principal de este proyecto ha sido montar una tienda online completa (**PrestaShop**) junto con su base de datos y un panel de administración, orquestando todos los servicios mediante contenedores con **Docker Compose**. Además, hemos implementado una capa de seguridad proxy inversa utilizando **HTTPS-Portal** para generar y renovar automáticamente los certificados SSL de *Let's Encrypt*.


## Explicando el `docker-compose.yml`

Toda la infraestructura se define y levanta a partir de un único archivo. En lugar de meter todos los servicios en la misma red, hemos diseñado una **arquitectura dividida** para maximizar la seguridad. 

Este es el código principal:

```yaml
version: '3.4'

services:
  mysql:
    image: mysql:8.0  # Fijamos versión para estabilidad
    command: --default-authentication-plugin=mysql_native_password # Para evitar problemas de conexión con PHP antiguos
    ports: 
      - 3306:3306
    environment: 
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes: 
      - mysql_data:/var/lib/mysql
    networks: 
      - backend-network
    restart: always
   
  phpmyadmin:
    image: phpmyadmin
    ports:
      - 8080:80
    environment: 
      - PMA_ARBITRARY=mysql
    networks: 
      - backend-network
      - frontend-network 
    restart: always
    depends_on: 
      - mysql

  prestashop:
    image: prestashop/prestashop:8.1
    environment: 
      - DB_SERVER=mysql
      - DB_USER=${DB_USER}
      - DB_PASSWD=${DB_PASSWD}
      - DB_PREFIX=${DB_PREFIX}
      - DB_NAME=${DB_NAME}
      - PS_DOMAIN=${PS_DOMAIN}
      - PS_LANGUAGE=${PS_LANGUAGE}
      - PS_COUNTRY=${PS_COUNTRY}
      - PS_ENABLE_SSL=1
      - PS_INSTALL_AUTO=1
      - ADMIN_MAIL=${ADMIN_MAIL}
      - ADMIN_PASSWD=${ADMIN_PASSWD}
    volumes:
      - prestashop_data:/var/www/html
    networks: 
      - backend-network
      - frontend-network
    restart: always
    depends_on: 
      - mysql

  https-portal:
    image: steveltn/https-portal:1
    ports:
      - 80:80
      - 443:443
    restart: always
    environment:
      # Mapea el dominio HTTPS exterior al servicio prestashop puerto 80 interior
      DOMAINS: '${DNS_DOMAIN_SECURE} -> http://prestashop:80' 
      STAGE: 'production' 
    networks:
      - frontend-network
    depends_on:
      - prestashop

volumes:
  mysql_data:
  prestashop_data:

networks: 
  frontend-network:
  backend-network:
```

### ¿Qué hace cada bloque exactamente?

*   **Aislamiento de la Base de Datos (`mysql`):** Lo aislamos en la red `backend-network`. Nadie desde el exterior puede tocar este contenedor directamente. Además, fijamos la versión a la **8.0** y usamos el plugin de contraseña nativa para evitar problemas de compatibilidad con versiones antiguas de PHP. Montamos un volumen (`mysql_data`) para **persistencia de datos**.
*   **Gestión Visual (`phpmyadmin`):** Conectado a ambas redes. Se comunica con MySQL por el backend y expone el puerto **8080** en el frontend para gestionar la base de datos de forma visual.
*   **El Corazón de la Tienda (`prestashop`):** Recibe variables de entorno (desde nuestro `.env`) para una **instalación desatendida**. Cuenta con su propio volumen persistente (`prestashop_data`) para guardar imágenes y módulos.
*   **El Guardaespaldas SSL (`https-portal`):** Pieza clave de la seguridad. Actúa como **proxy inverso**, exponiendo los puertos 80 y 443 a internet en la `frontend-network`. Intercepta el tráfico hacia nuestro dominio, negocia el **certificado SSL** y redirige el tráfico cifrado al puerto 80 interno del contenedor de PrestaShop.

## Resumen del Despliegue

### 1. Levantando los Contenedores

Con el `docker-compose.yml` y el `.env` (con las contraseñas y el dominio `miaumiaumiau.ddns.net`) preparados, iniciamos el entorno.

> *Despliegue del entorno con Docker Compose*

Con el comando `docker compose up -d` se descargan las imágenes y se inician las redes, volúmenes y contenedores en **segundo plano**.

![](images/docker%205.1%20cap1.png)

### 2. Verificación del Certificado SSL Automático

Revisando los logs del contenedor `https-portal`, confirmamos la negociación exitosa con Let's Encrypt, sin necesidad de configurar Certbot a mano.

> *Logs de la generación del certificado*

Ejecutamos `docker-compose logs https-portal` y verificamos la generación y firma automática del certificado para nuestro dominio.

![](images/docker%205.1%20cap%203.png)

### 3. Resultado Final: Tienda Securizada

Para comprobar el resultado, accedemos al dominio desde el navegador.

> *PrestaShop funcionando sobre HTTPS*

El frontend de PrestaShop es funcional y muestra el **candado de seguridad**, confirmando que el tráfico HTTPS y el certificado están correctamente implementados.

![](images/docker%205.1%20cap%202.png)