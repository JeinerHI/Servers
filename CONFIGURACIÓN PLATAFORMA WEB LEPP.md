### Installar Linux
Usaré WSL2 para este ejercicio, instalando la distrubución Ubuntu 22.04.5 LTS
>[!tip]
>Evita utilizar las últimas versiones de las distribuciones

### Instalar NginX
Empiezo con un update y upgrade para garantizar al puesta a punto del sistema operativo, después instalo Nginx:
```bash
sudo apt install nginx -y
```
Inicio y habilito apache
```bash
sudo service nginx start
```

**Nota importante sobre WSL2:** A diferencia de Linux nativo, en WSL2 no usamos `systemctl` sino `service`. Los servicios no se inician automáticamente al arrancar WSL.

#### Paso 2: Verificar que funciona

Abre tu navegador en Windows y ve a:
```
http://localhost
```
>[!note]
>Si tengo WAMP iniciado o aunque esté parado, lo he iniciado en algún momento durante la sesión, el "localhost" tratará de redirigirme a wamp, así que lo mejor es o ingresar por http://172.30.76.144/ >> La direccion ip de wsl << o reiniciar mi ordenador.

### Instalar y Configurar PostgreSQL server
```bash
sudo apt install postgresql postgresql-contrib -y
```
**Qué incluye esto?**
- `postgresql`: El servidor de base de datos
- `postgresql-contrib`: Módulos y extensiones adicionales útiles

Inicio el servicio
```bash
sudo service postgresql start
```
#### Paso 2: Configurar usuario y contraseña

**Explicación detallada:** PostgreSQL por defecto crea un usuario del sistema llamado `postgres`. Necesitamos configurarlo para desarrollo.

Accede al usuario postgres:
```bash
sudo -u postgres psql
```

Ahora estás dentro del prompt de PostgreSQL (`postgres=#`). Ejecuta:
```sql
ALTER USER postgres WITH PASSWORD 'postgres123';
```

Crea un usuario para desarrollo (opcional pero recomendado):
```sql
CREATE USER dev_user WITH PASSWORD 'dev123' CREATEDB;
```

**¿Qué hicimos?**
- Cambiamos la contraseña del superusuario `postgres`
- Creamos un usuario `dev_user` con permisos para crear bases de datos

Sal con:
```sql
\q
```

#### Paso 4: Habilitar autenticación por contraseña

Por defecto, PostgreSQL usa autenticación "peer" (solo desde el mismo usuario del sistema). Vamos a permitir conexiones con contraseña.

Edita el archivo de configuración:
````bash
sudo nano /etc/postgresql/*/main/pg_hba.conf
```

**Nota:** El asterisco (*) representa la versión de PostgreSQL instalada (probablemente 14 o 15).

Busca estas líneas cerca del final del archivo:
```
# "local" is for Unix domain socket connections only
local   all             all                                     peer
```

Cambia `peer` por `md5`:
```
local   all             all                                     md5
```

También busca:
```
host    all             all             127.0.0.1/32            scram-sha-256
```

Cámbialo a:
```
host    all             all             127.0.0.1/32            md5
````

Guarda (Ctrl+O, Enter, Ctrl+X).

#### Paso 5: Reiniciar PostgreSQL
```bash
sudo service postgresql restart
```

#### Paso 6: Probar acceso
```bash
psql -U postgres -h localhost
```

Ingresa la contraseña (`postgres123`). Si entras al prompt `postgres=#`, todo está bien.

Prueba también con el usuario de desarrollo:

bash

```bash
psql -U dev_user -h localhost -d postgres
```

Sal con `\q`

---

### Instalar y Configurar PHP para trabajar con PostgreSQL

#### Paso 1: Instalar extensión PHP para PostgreSQL
```bash
sudo apt install php-pgsql -y
```

#### Paso 2: Instalar PHP-FPM para Nginx

**Explicación importante:** Nginx no tiene un módulo PHP integrado como Apache. Usa PHP-FPM (FastCGI Process Manager), que es un procesador PHP independiente.
```bash
sudo apt install php-fpm -y
```

#### Paso 3: Verificar versión de PHP-FPM instalada

bash

```bash
php -v
```

Anota la versión (por ejemplo, `8.1`). La necesitarás para la configuración.

#### Paso 4: Iniciar PHP-FPM
```bash
sudo service php8.1-fpm start
```

**Nota:** Cambia `8.1` por tu versión si es diferente.

Verifica que está corriendo:
```bash
sudo service php8.1-fpm status
```


### Configurar Nginx para procesar PHP

#### Paso 1: Crear directorio para tu proyecto
```bash
sudo mkdir /var/www/lepp
sudo chown -R $USER:$USER /var/www/lepp
```

#### Paso 2: Configurar sitio en Nginx

Crea un archivo de configuración:
```bash
sudo nano /etc/nginx/sites-available/lepp
```

**Explicación detallada de la configuración:**

Pega este contenido (ajusta la versión de PHP si es necesaria):

nginx

```nginx
server {
    listen 80;
    listen [::]:80;
    
    root /var/www/lepp;
    index index.php index.html;
    
    server_name localhost;
    
    location / {
        try_files $uri $uri/ =404;
    }
    
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }
    
    location ~ /\.ht {
        deny all;
    }
}
```

**¿Qué hace cada parte?**

- `listen 80`: Escucha en puerto 80
- `root /var/www/lepp`: Carpeta raíz del sitio
- `index index.php`: Archivo índice por defecto
- `location ~ \.php$`: Procesa archivos .php con PHP-FPM
- `fastcgi_pass`: Socket donde PHP-FPM escucha
- `location ~ /\.ht`: Bloquea acceso a archivos .htaccess

Guarda y cierra.

#### Paso 3: Habilitar el sitio
```bash
sudo ln -s /etc/nginx/sites-available/lepp /etc/nginx/sites-enabled/
```
Esta línea de comando **activa** tu sitio web en el servidor Nginx. 

Aquí tienes el detalle técnico:

- **`ln -s`**: Crea un **enlace simbólico** (un acceso directo inteligente).
- **`/etc/nginx/sites-available/lepp`**: Es el archivo original donde escribiste la configuración de tu sitio.
- **`/etc/nginx/sites-enabled/`**: Es la carpeta que Nginx "lee" para saber qué sitios debe poner en marcha. 

¿Por qué se hace así?

Es una forma limpia de gestionar servidores: 

1. **Sites-available**: Es como un "almacén" de configuraciones. Puedes tener 10 sitios guardados aquí, pero no todos tienen que estar funcionando.
2. **Sites-enabled**: Solo contiene enlaces a los sitios que quieres que estén **al aire** en este momento.
#### Paso 4: Deshabilitar sitio por defecto (opcional)

bash

```bash
sudo rm /etc/nginx/sites-enabled/default
```

#### Paso 5: Verificar configuración de Nginx
```bash
sudo nginx -t
```

Debe decir: `syntax is ok` y `test is successful`.

#### Paso 6: Reiniciar Nginx
```bash
sudo service nginx restart
```

### Probar tu LEPP completo

#### Paso 1: Crear archivo PHP de prueba
```bash
nano /var/www/lepp/info.php
```
Incluye en siguiente contenido
```php
<?php
phpinfo();
?>
```
Guarda y cierra.

#### Paso 2: Verificar en navegador

Ve a:
```
http://localhost/info.php
````

Deberías ver la página de información de PHP. Busca la sección "pgsql" para confirmar que el módulo PostgreSQL está habilitado.

#### Paso 3: Probar conexión PHP-PostgreSQL
```bash
nano /var/www/lepp/test-db.php
```

Contenido:
```php
<?php
$host = "localhost";
$dbname = "postgres";
$user = "postgres";
$password = "postgres123";

try {
    $conexion = new PDO("pgsql:host=$host;dbname=$dbname", $user, $password);
    $conexion->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    
    echo "¡Conexión exitosa a PostgreSQL desde PHP!<br>";
    
    // Mostrar versión de PostgreSQL
    $version = $conexion->query('SELECT version()')->fetchColumn();
    echo "Versión: " . $version;
    
} catch(PDOException $e) {
    echo "Error de conexión: " . $e->getMessage();
}
?>
```

Accede a:
```
http://localhost/test-db.php
````

Si ves "¡Conexión exitosa..." y la versión de PostgreSQL → **¡LEPP funcionando perfectamente!** 🎉

---

## COMANDOS ÚTILES LEPP

### Iniciar todos los servicios:

bash

```bash
sudo service nginx start
sudo service postgresql start
sudo service php8.1-fpm start
```

### Ver estados:

bash

```bash
sudo service nginx status
sudo service postgresql status
sudo service php8.1-fpm status
```

### Reiniciar servicios:

bash

```bash
sudo service nginx restart
sudo service postgresql restart
sudo service php8.1-fpm restart
```

---

## GESTIÓN DE BASES DE DATOS PostgreSQL

### Comandos útiles desde psql:

bash

```bash
# Conectar
psql -U postgres -h localhost

# Dentro de psql:
\l                  # Listar bases de datos
\c nombre_db        # Conectar a una base de datos
\dt                 # Listar tablas
\du                 # Listar usuarios
\q                  # Salir
```

### Crear una base de datos de prueba:

bash

```bash
psql -U postgres -h localhost
```

sql

```sql
CREATE DATABASE mi_proyecto;
\c mi_proyecto
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100),
    email VARCHAR(100)
);
INSERT INTO usuarios (nombre, email) VALUES ('Juan', 'juan@example.com');
SELECT * FROM usuarios;
\q
```

---

## UBICACIONES IMPORTANTES LEPP

- **Archivos web:** `/var/www/lepp/`
- **Configuración Nginx:** `/etc/nginx/`
- **Sitios disponibles:** `/etc/nginx/sites-available/`
- **Sitios habilitados:** `/etc/nginx/sites-enabled/`
- **Logs Nginx:** `/var/log/nginx/`
- **Configuración PostgreSQL:** `/etc/postgresql/*/main/`
- **Datos PostgreSQL:** `/var/lib/postgresql/*/main/`
- **Configuración PHP-FPM:** `/etc/php/*/fpm/`