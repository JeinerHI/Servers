## Primeros pasos
Empezando se decide con qué plataforma se quiere trabajar, la plataforma es el conjunto de servicios que irán montados en el servidor para servir a la aplicación. Para el ejemplo del ejercicio usaré [[LAMP]] montado sobre mi máquina de WSL2 con la distribución Ubuntu-22.04
>[!tip]
>Evita utilizar las últimas versiones de las distribuciones

### Instalar Apache2
Empiezo con un update y upgrade para garantizar al puesta a punto del sistema operativo, después instalo apache:
```bash
sudo apt install apache2 -y
```
Inicio y habilito apache
```bash
sudo service apache2 start
```

**Nota importante sobre WSL2:** A diferencia de Linux nativo, en WSL2 no usamos `systemctl` sino `service`. Los servicios no se inician automáticamente al arrancar WSL.

#### Paso 2: Verificar que funciona

Abre tu navegador en Windows y ve a:
```
http://localhost
```
>[!note]
>Si tengo WAMP iniciado o aunque esté parado, lo he iniciado en algún momento durante la sesión, el "localhost" tratará de redirigirme a wamp, así que lo mejor es o ingresar por http://172.22.143.8/ >> La direccion ip de wsl << o reiniciar mi ordenador.

### Instalar MySQL server
```bash
sudo apt install mysql-server -y
```
Inicio el servicio MySQL
```bash
sudo service mysql start
```
#### Paso 2: Configuración básica de seguridad
```bash
sudo mysql_secure_installation
```

**Aquí viene una parte importante:**

1. Te preguntará si quieres configurar el plugin de validación de contraseñas → Responde `N` (no, para simplificar)
2. Establecer contraseña para root → Escribe una contraseña simple (ej: `root123`)
3. Remover usuarios anónimos → `Y`
4. Deshabilitar login root remoto → `Y`
5. Remover base de datos de prueba → `Y`
6. Recargar tablas de privilegios → `Y`

#### Paso 3: Configurar autenticación para root

Por defecto, MySQL en Ubuntu usa `auth_socket` para root. Vamos a cambiarlo para poder acceder con contraseña:
```bash
sudo mysql
```

Dentro de MySQL ejecuta:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'root123';
FLUSH PRIVILEGES;
EXIT;
```

**¿Por qué hacemos esto?** Para que puedas acceder a MySQL con `mysql -u root -p` usando tu contraseña, que es más cómodo para desarrollo.
#### Paso 3: Probar acceso

```bash
mysql -u root -p
```

Ingresa tu contraseña. Si entras al prompt `mysql>`, todo está bien. Sal con `EXIT;`

### Instalación de PHP (la "P" de LAMP)

#### Instalar PHP y módulos necesarios
```bash
sudo apt install php libapache2-mod-php php-mysql -y
```

**¿Qué instalamos?**

- `php`: El intérprete de PHP
- `libapache2-mod-php`: Módulo para que Apache procese archivos PHP
- `php-mysql`: Extensión para que PHP se conecte con MySQL

#### Paso 2: Verificar la instalación
```bash
php -v
```

Deberías ver algo como `PHP 8.1.x`

#### Paso 3: Configurar Apache para priorizar PHP

Editamos la configuración de Apache:
```bash
sudo nano /etc/apache2/mods-enabled/dir.conf
```

**Explicación detallada:** Verás una línea que empieza con `DirectoryIndex`. Este parámetro define qué archivo busca Apache por defecto en un directorio. Queremos que `index.php` tenga prioridad sobre `index.html`.

Modifica la línea para que quede así:
```apache
DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
```

Guarda con `Ctrl+O`, Enter, y sal con `Ctrl+X`.

#### Paso 4: Reiniciar Apache
```bash
sudo service apache2 restart
```

## FASE 6: Probar tu LAMP completo

### Paso 18: Crear un archivo PHP de prueba

bash

```bash
sudo nano /var/www/html/info.php
```

**Nota sobre permisos:** `/var/www/html/` es el directorio raíz web de Apache. Necesitas `sudo` para escribir aquí.

Escribe este contenido:

php

````php
<?php
phpinfo();
?>
```

Guarda y cierra (Ctrl+O, Enter, Ctrl+X).

### Paso 19: Verificar en el navegador

Ve a:
```
http://localhost/info.php
````

Deberías ver una página con toda la información de PHP, incluyendo que MySQL está habilitado.

### Paso 20: Probar conexión PHP-MySQL

Crea otro archivo de prueba:

bash

```bash
sudo nano /var/www/html/test-db.php
```

Contenido:

php

````php
<?php
$conexion = new mysqli("localhost", "root", "root123", "mysql");

if ($conexion->connect_error) {
    die("Error de conexión: " . $conexion->connect_error);
}

echo "¡Conexión exitosa a MySQL desde PHP!";
$conexion->close();
?>
```

Accede a:
```
http://localhost/test-db.php
````

Si ves "¡Conexión exitosa a MySQL desde PHP!" → **¡Enhorabuena, tu LAMP está funcionando!** 🚀