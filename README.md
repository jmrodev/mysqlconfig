# Guía Definitiva: Configuración Completa del Almacenamiento de phpMyAdmin

## Introducción

Este documento es una guía paso a paso, diseñada para principiantes, que soluciona los problemas de configuración más comunes de phpMyAdmin después de su instalación. Si ves advertencias como **"El almacenamiento de configuración phpMyAdmin no está completamente configurado"** o **"El directorio TempDir es inaccesible"**, esta guía es para ti.

Seguiremos un proceso robusto para:
1.  Crear y configurar una base de datos de control para phpMyAdmin.
2.  Solucionar problemas de permisos de directorios temporales (específicamente en Arch Linux).
3.  Configurar correctamente el archivo `config.inc.php` con claves seguras y los parámetros necesarios.
4.  Sincronizar las contraseñas para evitar el temido error de **"Access denied for user 'pma'"**.

El objetivo es dejar tu instalación de phpMyAdmin limpia, segura y 100% funcional.

---

## Paso 1: El Script de la Base de Datos (`phpmyadmin_config.sql`)

Primero, necesitamos un script SQL que cree la base de datos `phpmyadmin` y todas las tablas `pma__*` que habilitan las funciones extendidas. También crea un usuario de control llamado `pma`. El contenido completo del archivo `phpmyadmin_config.sql` que debes usar es el siguiente.

```sql
-- --------------------------------------------------------
-- SQL Commands to set up the pmadb as described in the documentation.
--
-- This script creates the user 'pma' with a default password 'pmapass'.
-- It's strongly recommended to change this password after setup.
--
-- This user "pma" must be defined in config.inc.php (controluser/controlpass)
--
-- Please don't forget to set up the tablenames in config.inc.php
--

-- --------------------------------------------------------

--
-- Database : `phpmyadmin`
--
CREATE DATABASE IF NOT EXISTS `phpmyadmin`
  DEFAULT CHARACTER SET utf8 COLLATE utf8_bin;
USE phpmyadmin;

-- (Aquí siguen todas las definiciones de tablas `pma__*`. A continuación, un ejemplo)

--
-- Table structure for table `pma__bookmark`
--
CREATE TABLE IF NOT EXISTS `pma__bookmark` (
  `id` int(11) NOT NULL auto_increment,
  `dbase` varchar(255) NOT NULL default '',
  `user` varchar(255) NOT NULL default '',
  `label` varchar(255) COLLATE utf8_general_ci NOT NULL default '',
  `query` text NOT NULL,
  PRIMARY KEY  (`id`)
)
  COMMENT='Bookmarks'
  DEFAULT CHARACTER SET utf8 COLLATE utf8_bin;

-- ... (y el resto de las más de 20 tablas) ...

--
-- Create and grant privileges to the control user
--
CREATE USER IF NOT EXISTS 'pma'@'localhost' IDENTIFIED BY 'pmapass';
GRANT SELECT, INSERT, UPDATE, DELETE ON `phpmyadmin`.* TO 'pma'@'localhost';
```

**Acción:** Ejecuta este script desde la raíz de tu proyecto para crear todo lo necesario en MariaDB.

```bash
mariadb -u root -p < backend/database/phpmyadmin_config.sql
```
Te pedirá la contraseña de `root` de tu base de datos. Si la base de datos `phpmyadmin` ya existe, no te preocupes, el script está diseñado para no fallar.

---

## Paso 2: Solucionar el Error de `TempDir` (Específico de Arch Linux)

phpMyAdmin necesita un directorio para guardar archivos temporales y mejorar el rendimiento. La advertencia **"El $cfg['TempDir'] (...) es inaccesible"** se soluciona creando este directorio y dándole los permisos correctos al servidor web.

En Arch Linux, el usuario del servidor web es `http`.

```bash
# 1. Crear el directorio y sus padres si no existen
sudo mkdir -p /usr/share/webapps/phpMyAdmin/tmp/

# 2. Asignar la propiedad al usuario y grupo 'http'
sudo chown http:http /usr/share/webapps/phpMyAdmin/tmp/

# 3. Asignar los permisos correctos
sudo chmod 755 /usr/share/webapps/phpMyAdmin/tmp/
```

---

## Paso 3: Configurar el Archivo `config.inc.php`

Este es el paso más importante. Aquí le decimos a phpMyAdmin cómo conectarse a su base de datos de control y establecemos claves de seguridad.

1.  **Localiza tu archivo:** En Arch Linux, se encuentra en `/etc/webapps/phpmyadmin/config.inc.php`.

2.  **Reemplaza todo el contenido del archivo** con la siguiente plantilla genérica. Esta plantilla está diseñada para ser copiada y pegada, pero requiere que completes dos valores.

    **Acción 1: Genera tu clave `blowfish_secret`**
    Esta clave cifra las cookies y debe ser una cadena aleatoria de 32 caracteres. **No uses la del ejemplo.** Genera la tuya con este comando y pégala en el archivo:
    ```bash
    openssl rand -base64 32 | head -c 32
    ```

    **Acción 2: Elige una contraseña para `controlpass`**
    Elige una contraseña segura para el usuario `pma`. Esta es la contraseña que sincronizaremos en el siguiente paso.

    ```php
    <?php
    /**
     * Plantilla de Configuración Genérica y Segura para phpMyAdmin
     */
    declare(strict_types=1);

    /**
     * Clave secreta para el cifrado de cookies.
     * ¡DEBES REEMPLAZAR ESTE VALOR POR EL TUYO PROPIO!
     */
    $cfg['blowfish_secret'] = 'REEMPLAZA_CON_TU_CLAVE_DE_32_CARACTERES';

    /**
     * Configuración del Servidor Principal
     */
    $i = 1;
    $cfg['Servers'][$i]['auth_type'] = 'cookie';
    $cfg['Servers'][$i]['host'] = 'localhost';
    $cfg['Servers'][$i]['compress'] = false;
    $cfg['Servers'][$i]['AllowNoPassword'] = false;

    /**
     * Configuración del Almacenamiento Extendido
     */
    $cfg['Servers'][$i]['controluser'] = 'pma';
    $cfg['Servers'][$i]['controlpass'] = 'TU_CONTRASEÑA_SEGURA_PARA_PMA'; // ¡REEMPLAZA ESTO!

    $cfg['Servers'][$i]['pmadb'] = 'phpmyadmin';
    $cfg['Servers'][$i]['bookmarktable'] = 'pma__bookmark';
    $cfg['Servers'][$i]['relation'] = 'pma__relation';
    $cfg['Servers'][$i]['table_info'] = 'pma__table_info';
    $cfg['Servers'][$i]['table_coords'] = 'pma__table_coords';
    $cfg['Servers'][$i]['pdf_pages'] = 'pma__pdf_pages';
    $cfg['Servers'][$i]['column_info'] = 'pma__column_info';
    $cfg['Servers'][$i]['history'] = 'pma__history';
    $cfg['Servers'][$i]['table_uiprefs'] = 'pma__table_uiprefs';
    $cfg['Servers'][$i]['tracking'] = 'pma__tracking';
    $cfg['Servers'][$i]['userconfig'] = 'pma__userconfig';
    $cfg['Servers'][$i]['recent'] = 'pma__recent';
    $cfg['Servers'][$i]['favorite'] = 'pma__favorite';
    $cfg['Servers'][$i]['users'] = 'pma__users';
    $cfg['Servers'][$i]['usergroups'] = 'pma__usergroups';
    $cfg['Servers'][$i]['navigationhiding'] = 'pma__navigationhiding';
    $cfg['Servers'][$i]['savedsearches'] = 'pma__savedsearches';
    $cfg['Servers'][$i]['central_columns'] = 'pma__central_columns';
    $cfg['Servers'][$i]['designer_settings'] = 'pma__designer_settings';
    $cfg['Servers'][$i]['export_templates'] = 'pma__export_templates';

    /**
     * Directorios
     */
    $cfg['UploadDir'] = '';
    $cfg['SaveDir'] = '';
    $cfg['TempDir'] = '/usr/share/webapps/phpMyAdmin/tmp/';
    ?>
    ```

---

## Paso 4: Sincronizar la Contraseña (La Causa del Error "Access Denied")

Este es el paso final y el que soluciona la mayoría de los problemas. Debemos asegurarnos de que la contraseña en la base de datos para el usuario `pma` es **exactamente la misma** que la que pusiste en `$cfg['Servers'][$i]['controlpass']`.

1.  **Conéctate a MariaDB como `root`:**
    ```bash
    mariadb -u root -p
    ```
2.  **Cambia la contraseña del usuario `pma`:**
    Usa el siguiente comando, reemplazando `TU_CONTRASEÑA_SEGURA_PARA_PMA` por la misma contraseña que elegiste en el paso anterior.
    ```sql
    ALTER USER 'pma'@'localhost' IDENTIFIED BY 'TU_CONTRASEÑA_SEGURA_PARA_PMA';
    ```
3.  **Aplica los cambios y sal:**
    ```sql
    FLUSH PRIVILEGES;
    EXIT;
    ```

---

## Paso 5: Reiniciar y Verificar

Para que todos los cambios se apliquen, reinicia el servidor web.

```bash
# Comando para Arch Linux con Apache
sudo systemctl restart httpd
```

Ahora, cierra sesión completamente en phpMyAdmin, cierra la pestaña del navegador, ábrela de nuevo e inicia sesión. Todas las advertencias y errores de configuración deberían haber desaparecido. 