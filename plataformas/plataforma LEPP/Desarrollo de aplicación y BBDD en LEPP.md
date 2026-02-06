## BLOQUE 1: BASE DE DATOS

### Paso 1: Crear base de datos y usuario
```bash
sudo -u postgres psql
```

Dentro de PostgreSQL 
```sql
-- Crear base de datos
CREATE DATABASE gestion_proyectos;

-- Crear usuario con contraseña
CREATE USER app_user WITH PASSWORD 'app123';

-- Dar permisos solo a esta base de datos
GRANT ALL PRIVILEGES ON DATABASE gestion_proyectos TO app_user;

-- Conectar a la base de datos
\c gestion_proyectos

-- Dar permisos en el schema public
GRANT ALL ON SCHEMA public TO app_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO app_user;

-- Crear tabla proyectos
CREATE TABLE proyectos (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    descripcion TEXT,
    fecha_inicio DATE,
    estado VARCHAR(50) DEFAULT 'En Planificación',
    presupuesto DECIMAL(10,2),
    fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insertar datos de prueba
INSERT INTO proyectos (nombre, descripcion, fecha_inicio, estado, presupuesto) VALUES
('Proyecto Web', 'Desarrollo de sitio corporativo', '2024-01-15', 'En Desarrollo', 15000.00),
('App Móvil', 'Aplicación para inventario', '2024-02-01', 'En Planificación', 25000.00),
('Sistema ERP', 'Implementación ERP empresarial', '2023-11-10', 'Completado', 50000.00);

-- Verificar
SELECT * FROM proyectos;
```

## BLOQUE 2: EXTENSIONES PHP

```bash
# Instalar extensiones que faltan
sudo apt install php-json php-xml php-mbstring php-curl -y

# Reiniciar PHP-FPM
sudo service php8.1-fpm restart

# Verificar extensiones
php -m | grep -E "json|xml|mbstring|pgsql"
```

Deberías ver las 4 extensiones listadas.

---

## BLOQUE 3: CREAR ESTRUCTURA DEL PROYECTO

### Paso 1: Crear carpetas
```bash
# Crear directorio del proyecto
sudo mkdir -p /var/www/gestion-proyectos
sudo chown -R $USER:$USER /var/www/gestion-proyectos
cd /var/www/gestion-proyectos
```

### Paso 2: Crear archivo de configuración DB

```bash
nano config.php
```

Pega esto:

php

```php
<?php
/**
 * Configuración de conexión a PostgreSQL
 * Base de datos: gestion_proyectos
 */

define('DB_HOST', 'localhost');
define('DB_NAME', 'gestion_proyectos');
define('DB_USER', 'app_user');
define('DB_PASS', 'app123');
define('DB_PORT', '5432');

/**
 * Función para obtener conexión PDO
 * @return PDO
 */
function getDB() {
    try {
        $dsn = "pgsql:host=" . DB_HOST . ";port=" . DB_PORT . ";dbname=" . DB_NAME;
        $pdo = new PDO($dsn, DB_USER, DB_PASS);
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
        return $pdo;
    } catch(PDOException $e) {
        die("Error de conexión: " . $e->getMessage());
    }
}
?>
```

### Paso 3: Crear index.php (CRUD - Listar)

```bash
nano index.php
```

Pega esto:

php

```php
<?php
/**
 * Página principal - Listado de proyectos
 */
require_once 'config.php';

// Obtener todos los proyectos
try {
    $db = getDB();
    $stmt = $db->query("SELECT * FROM proyectos ORDER BY id DESC");
    $proyectos = $stmt->fetchAll();
} catch(PDOException $e) {
    $error = "Error al obtener proyectos: " . $e->getMessage();
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gestión de Proyectos</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>📋 Sistema de Gestión de Proyectos</h1>
        
        <?php if(isset($error)): ?>
            <div class="error"><?= $error ?></div>
        <?php endif; ?>
        
        <div class="actions">
            <a href="crear.php" class="btn btn-success">➕ Nuevo Proyecto</a>
            <a href="info.php" class="btn btn-info">ℹ️ PHP Info</a>
        </div>
        
        <table>
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Nombre</th>
                    <th>Descripción</th>
                    <th>Fecha Inicio</th>
                    <th>Estado</th>
                    <th>Presupuesto</th>
                    <th>Acciones</th>
                </tr>
            </thead>
            <tbody>
                <?php if(empty($proyectos)): ?>
                    <tr><td colspan="7">No hay proyectos registrados</td></tr>
                <?php else: ?>
                    <?php foreach($proyectos as $proyecto): ?>
                        <tr>
                            <td><?= $proyecto['id'] ?></td>
                            <td><?= htmlspecialchars($proyecto['nombre']) ?></td>
                            <td><?= htmlspecialchars($proyecto['descripcion']) ?></td>
                            <td><?= $proyecto['fecha_inicio'] ?></td>
                            <td><span class="badge"><?= $proyecto['estado'] ?></span></td>
                            <td>$<?= number_format($proyecto['presupuesto'], 2) ?></td>
                            <td class="actions-cell">
                                <a href="editar.php?id=<?= $proyecto['id'] ?>" class="btn-small btn-warning">✏️</a>
                                <a href="eliminar.php?id=<?= $proyecto['id'] ?>" 
                                   class="btn-small btn-danger"
                                   onclick="return confirm('¿Eliminar este proyecto?')">🗑️</a>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                <?php endif; ?>
            </tbody>
        </table>
    </div>
</body>
</html>
```

### Paso 4: Crear crear.php

bash

```bash
nano crear.php
```

```php
<?php
/**
 * Formulario para crear nuevo proyecto
 */
require_once 'config.php';

if($_SERVER['REQUEST_METHOD'] == 'POST') {
    $nombre = $_POST['nombre'];
    $descripcion = $_POST['descripcion'];
    $fecha_inicio = $_POST['fecha_inicio'];
    $estado = $_POST['estado'];
    $presupuesto = $_POST['presupuesto'];
    
    try {
        $db = getDB();
        $sql = "INSERT INTO proyectos (nombre, descripcion, fecha_inicio, estado, presupuesto) 
                VALUES (:nombre, :descripcion, :fecha_inicio, :estado, :presupuesto)";
        $stmt = $db->prepare($sql);
        $stmt->execute([
            ':nombre' => $nombre,
            ':descripcion' => $descripcion,
            ':fecha_inicio' => $fecha_inicio,
            ':estado' => $estado,
            ':presupuesto' => $presupuesto
        ]);
        
        header('Location: index.php');
        exit;
    } catch(PDOException $e) {
        $error = "Error al crear proyecto: " . $e->getMessage();
    }
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Crear Proyecto</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>➕ Crear Nuevo Proyecto</h1>
        
        <?php if(isset($error)): ?>
            <div class="error"><?= $error ?></div>
        <?php endif; ?>
        
        <form method="POST">
            <div class="form-group">
                <label>Nombre del Proyecto:</label>
                <input type="text" name="nombre" required>
            </div>
            
            <div class="form-group">
                <label>Descripción:</label>
                <textarea name="descripcion" rows="4"></textarea>
            </div>
            
            <div class="form-group">
                <label>Fecha de Inicio:</label>
                <input type="date" name="fecha_inicio" required>
            </div>
            
            <div class="form-group">
                <label>Estado:</label>
                <select name="estado">
                    <option>En Planificación</option>
                    <option>En Desarrollo</option>
                    <option>En Pruebas</option>
                    <option>Completado</option>
                    <option>Cancelado</option>
                </select>
            </div>
            
            <div class="form-group">
                <label>Presupuesto ($):</label>
                <input type="number" step="0.01" name="presupuesto" required>
            </div>
            
            <div class="actions">
                <button type="submit" class="btn btn-success">💾 Guardar</button>
                <a href="index.php" class="btn btn-secondary">❌ Cancelar</a>
            </div>
        </form>
    </div>
</body>
</html>
```

### Paso 5: Crear editar.php

```bash
nano editar.php
```

```php
<?php
/**
 * Formulario para editar proyecto existente
 */
require_once 'config.php';

$id = $_GET['id'] ?? 0;

// Obtener datos del proyecto
try {
    $db = getDB();
    $stmt = $db->prepare("SELECT * FROM proyectos WHERE id = :id");
    $stmt->execute([':id' => $id]);
    $proyecto = $stmt->fetch();
    
    if(!$proyecto) {
        header('Location: index.php');
        exit;
    }
} catch(PDOException $e) {
    die("Error: " . $e->getMessage());
}

// Procesar actualización
if($_SERVER['REQUEST_METHOD'] == 'POST') {
    try {
        $sql = "UPDATE proyectos SET 
                nombre = :nombre,
                descripcion = :descripcion,
                fecha_inicio = :fecha_inicio,
                estado = :estado,
                presupuesto = :presupuesto
                WHERE id = :id";
        
        $stmt = $db->prepare($sql);
        $stmt->execute([
            ':nombre' => $_POST['nombre'],
            ':descripcion' => $_POST['descripcion'],
            ':fecha_inicio' => $_POST['fecha_inicio'],
            ':estado' => $_POST['estado'],
            ':presupuesto' => $_POST['presupuesto'],
            ':id' => $id
        ]);
        
        header('Location: index.php');
        exit;
    } catch(PDOException $e) {
        $error = "Error al actualizar: " . $e->getMessage();
    }
}
?>
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Editar Proyecto</title>
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <div class="container">
        <h1>✏️ Editar Proyecto #<?= $id ?></h1>
        
        <?php if(isset($error)): ?>
            <div class="error"><?= $error ?></div>
        <?php endif; ?>
        
        <form method="POST">
            <div class="form-group">
                <label>Nombre:</label>
                <input type="text" name="nombre" value="<?= htmlspecialchars($proyecto['nombre']) ?>" required>
            </div>
            
            <div class="form-group">
                <label>Descripción:</label>
                <textarea name="descripcion" rows="4"><?= htmlspecialchars($proyecto['descripcion']) ?></textarea>
            </div>
            
            <div class="form-group">
                <label>Fecha de Inicio:</label>
                <input type="date" name="fecha_inicio" value="<?= $proyecto['fecha_inicio'] ?>" required>
            </div>
            
            <div class="form-group">
                <label>Estado:</label>
                <select name="estado">
                    <option <?= $proyecto['estado']=='En Planificación'?'selected':'' ?>>En Planificación</option>
                    <option <?= $proyecto['estado']=='En Desarrollo'?'selected':'' ?>>En Desarrollo</option>
                    <option <?= $proyecto['estado']=='En Pruebas'?'selected':'' ?>>En Pruebas</option>
                    <option <?= $proyecto['estado']=='Completado'?'selected':'' ?>>Completado</option>
                    <option <?= $proyecto['estado']=='Cancelado'?'selected':'' ?>>Cancelado</option>
                </select>
            </div>
            
            <div class="form-group">
                <label>Presupuesto ($):</label>
                <input type="number" step="0.01" name="presupuesto" value="<?= $proyecto['presupuesto'] ?>" required>
            </div>
            
            <div class="actions">
                <button type="submit" class="btn btn-success">💾 Actualizar</button>
                <a href="index.php" class="btn btn-secondary">❌ Cancelar</a>
            </div>
        </form>
    </div>
</body>
</html>
```

### Paso 6: Crear eliminar.php

```bash
nano eliminar.php
```

```php
<?php
/**
 * Eliminar proyecto
 */
require_once 'config.php';

$id = $_GET['id'] ?? 0;

try {
    $db = getDB();
    $stmt = $db->prepare("DELETE FROM proyectos WHERE id = :id");
    $stmt->execute([':id' => $id]);
} catch(PDOException $e) {
    die("Error al eliminar: " . $e->getMessage());
}

header('Location: index.php');
exit;
?>
```

### Paso 7: Crear info.php

bash

```bash
nano info.php
```

php

```php
<?php
/**
 * Información del sistema PHP
 */
phpinfo();
?>
```

### Paso 8: Crear styles.css

bash

```bash
nano styles.css
```

css

```css
/* Estilos para Sistema de Gestión de Proyectos */

* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    padding: 20px;
}

.container {
    max-width: 1200px;
    margin: 0 auto;
    background: white;
    padding: 30px;
    border-radius: 10px;
    box-shadow: 0 10px 40px rgba(0,0,0,0.2);
}

h1 {
    color: #333;
    margin-bottom: 30px;
    text-align: center;
}

.actions {
    margin-bottom: 20px;
    display: flex;
    gap: 10px;
}

.btn {
    padding: 10px 20px;
    text-decoration: none;
    border-radius: 5px;
    display: inline-block;
    font-weight: bold;
    transition: all 0.3s;
    border: none;
    cursor: pointer;
}

.btn-success {
    background: #28a745;
    color: white;
}

.btn-success:hover {
    background: #218838;
}

.btn-info {
    background: #17a2b8;
    color: white;
}

.btn-warning {
    background: #ffc107;
    color: #333;
}

.btn-danger {
    background: #dc3545;
    color: white;
}

.btn-secondary {
    background: #6c757d;
    color: white;
}

.btn-small {
    padding: 5px 10px;
    font-size: 14px;
}

table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 20px;
}

th, td {
    padding: 12px;
    text-align: left;
    border-bottom: 1px solid #ddd;
}

th {
    background: #667eea;
    color: white;
}

tr:hover {
    background: #f5f5f5;
}

.badge {
    padding: 5px 10px;
    border-radius: 15px;
    background: #667eea;
    color: white;
    font-size: 12px;
}

.actions-cell {
    display: flex;
    gap: 5px;
}

.form-group {
    margin-bottom: 20px;
}

.form-group label {
    display: block;
    margin-bottom: 5px;
    font-weight: bold;
    color: #333;
}

.form-group input,
.form-group textarea,
.form-group select {
    width: 100%;
    padding: 10px;
    border: 1px solid #ddd;
    border-radius: 5px;
    font-size: 14px;
}

.error {
    background: #f8d7da;
    color: #721c24;
    padding: 15px;
    border-radius: 5px;
    margin-bottom: 20px;
    border: 1px solid #f5c6cb;
}
```

## BLOQUE 4: VIRTUALHOST NGINX

```bash
sudo nano /etc/nginx/sites-available/gestion-proyectos
```

Añadir el siguiente contenido al script:
```nginx
# VirtualHost para Gestión de Proyectos
# Configuración con SSL/TLS

# Redirección HTTP a HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name gestion-proyectos.local;
    
    return 301 https://$server_name$request_uri;
}

# Servidor HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    # Directorio raíz del proyecto
    root /var/www/gestion-proyectos;
    index index.php index.html;
    
    server_name gestion-proyectos.local;
    
    # Certificados SSL (autofirmados)
    ssl_certificate /etc/ssl/selfsigned/localhost-san.crt;
    ssl_certificate_key /etc/ssl/selfsigned/localhost-san.key;
    
    # Configuración SSL segura
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Headers de seguridad
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Logs específicos del proyecto
    access_log /var/log/nginx/gestion-proyectos-access.log;
    error_log /var/log/nginx/gestion-proyectos-error.log;
    
    # Configuración de archivos PHP
    location / {
        try_files $uri $uri/ =404;
    }
    
    # Procesamiento PHP con PHP-FPM
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # Denegar acceso a archivos ocultos
    location ~ /\.ht {
        deny all;
    }
    
    # Denegar acceso a archivos de configuración
    location ~ /\.git {
        deny all;
    }
}
```

Habilitar sitio:

```bash
sudo ln -s /etc/nginx/sites-available/gestion-proyectos /etc/nginx/sites-enabled/
sudo nginx -t
sudo service nginx restart
```

** PORBLEME ENCONTRADO ** los certificados self-signed no estaban generados en la ubicación correcta. Procedo con la siguiente solución:

```bash
# Crear directorio
sudo mkdir -p /etc/ssl/selfsigned

# Generar certificados autofirmados
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/selfsigned/localhost-san.key \
  -out /etc/ssl/selfsigned/localhost-san.crt \
  -subj "/C=ES/ST=Madrid/L=Madrid/O=Dev/OU=IT/CN=gestion-proyectos.local"

# Verificar que existen
ls -l /etc/ssl/selfsigned/
```

Añadir al hosts de Windows. En **PowerShell como administrador**:

```powershell
Add-Content C:\Windows\System32\drivers\etc\hosts "127.0.0.1 gestion-proyectos.local"
```

---

## BLOQUE 6: FIREWALL

```bash
# Instalar UFW
sudo apt install ufw -y

# Permitir SSH (¡IMPORTANTE!)
sudo ufw allow 22/tcp

# Permitir HTTP y HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Permitir puertos alternativos (Apache)
sudo ufw allow 8080/tcp
sudo ufw allow 8443/tcp

# Activar firewall
sudo ufw --force enable

# Ver estado
sudo ufw status
```

---

## BLOQUE 7: PERMISOS

```bash
# Ajustar propietario
sudo chown -R www-data:www-data /var/www/gestion-proyectos

# Permisos de directorios
sudo find /var/www/gestion-proyectos -type d -exec chmod 755 {} \;

# Permisos de archivos
sudo find /var/www/gestion-proyectos -type f -exec chmod 644 {} \;
```

---

## BLOQUE 8: PROBAR TODO

Abre navegador:
```
https://gestion-proyectos.local
````

Acepta certificado autofirmado.

**Prueba:**

1. ¿Ves la lista de proyectos? ✅
2. Click "Nuevo Proyecto" → Crear uno ✅
3. Editar un proyecto ✅
4. Eliminar un proyecto ✅
5. Ir a `https://gestion-proyectos.local/info.php` ✅

** Problema Encontrado: ** 
![[Pasted image 20260206010946.png]]

El usuario `app_user` no tiene permisos sobre la secuencia. Se soluciona a continuación:

```bash
sudo -u postgres psql -d gestion_proyectos
```

Dentro de PostgreSQL ejecuta:

```sql
-- Dar permisos sobre la secuencia
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO app_user;

-- Verificar permisos
\dp proyectos_id_seq

\q
```