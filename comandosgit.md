# 📚 Guía de Comandos Git - Resumen Completo

---

## ⚙️ CONFIGURACIÓN INICIAL

```bash
git config --global user.name "Tu Nombre"
# Configura tu nombre para todos los repositorios

git config --global user.email "tu@email.com"
# Configura tu email para todos los repositorios

git config --global init.defaultBranch main
# Establece "main" como nombre de rama por defecto

git config --global core.editor "code --wait"
# Configura VSCode como editor por defecto

git config --global color.ui auto
# Activa colores en la salida de Git

git config --list
# Muestra toda la configuración actual

git config --list --show-origin
# Muestra configuración y desde qué archivo proviene

git config --global --list
# Muestra solo configuración global

git config --local --list
# Muestra solo configuración del proyecto actual

git config user.name
# Muestra tu nombre configurado

git config user.email
# Muestra tu email configurado
```

---

## 🆕 INICIALIZAR REPOSITORIO

```bash
git init
# Crea un nuevo repositorio Git en la carpeta actual

git clone https://github.com/usuario/repo.git
# Clona (descarga) un repositorio remoto a tu PC
```

---

## 📊 CONSULTAR ESTADO

```bash
git status
# Muestra el estado actual: rama, archivos modificados, etc.

git branch
# Lista todas las ramas locales y marca en cuál estás

git branch -a
# Lista todas las ramas (locales y remotas)

git branch -r
# Lista solo las ramas remotas

git branch -vv
# Muestra ramas locales con info de seguimiento remoto

git log --oneline
# Muestra historial de commits de forma resumida

git remote -v
# Muestra los repositorios remotos configurados y sus URLs
```

---

## 📝 GUARDAR CAMBIOS (COMMIT)

```bash
git add .
# Agrega TODOS los archivos modificados al staging

git add archivo.txt
# Agrega un archivo específico al staging

git commit -m "Mensaje descriptivo"
# Guarda los cambios en el historial con un mensaje

git commit -m "Initial commit"
# Primer commit típico de un proyecto nuevo
```

---

## 🌿 GESTIÓN DE RAMAS

### Crear y cambiar ramas

```bash
git checkout -b nombre-rama
# Crea una nueva rama Y cambia a ella

git switch -c nombre-rama
# Versión moderna: crea rama y cambia a ella

git checkout nombre-rama
# Cambia a una rama existente

git switch nombre-rama
# Versión moderna: cambia a una rama existente
```

### Renombrar ramas

```bash
git branch -m nuevo-nombre
# Renombra la rama actual

git branch -m viejo-nombre nuevo-nombre
# Renombra una rama específica (desde otra rama)
```

### Eliminar ramas

```bash
git branch -d nombre-rama
# Elimina una rama local (solo si está fusionada)

git branch -D nombre-rama
# Fuerza la eliminación de una rama local

git push origin --delete nombre-rama
# Elimina una rama del repositorio remoto (GitHub)
```

---

## 🔗 GESTIÓN DE REMOTOS

```bash
git remote add origin https://github.com/usuario/repo.git
# Conecta tu repositorio local con uno remoto

git remote set-url origin https://github.com/usuario/nuevo-repo.git
# Cambia la URL del remoto existente

git remote remove origin
# Elimina la conexión con el remoto

git remote -v
# Muestra las URLs de los remotos configurados
```

---

## 🔄 SINCRONIZACIÓN (PULL/PUSH)

### Descargar cambios (Pull)

```bash
git pull
# Descarga cambios de la rama remota vinculada

git pull origin nombre-rama
# Descarga cambios de una rama específica del remoto

git fetch
# Descarga info de ramas remotas SIN fusionar

git fetch origin
# Actualiza información del remoto "origin"

git fetch --prune
# Elimina referencias a ramas remotas ya borradas
```

### Subir cambios (Push)

```bash
git push
# Sube commits a la rama remota vinculada

git push -u origin nombre-rama
# Sube rama Y la vincula con el remoto (primera vez)

git push origin nombre-rama
# Sube commits a una rama específica del remoto
```

---

## 🔀 FLUJO DE TRABAJO COMPLETO

### Iniciar proyecto nuevo

```bash
git init
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin https://github.com/usuario/repo.git
git push -u origin main
```

### Trabajo diario

```bash
# Al empezar el día
git pull

# Trabajar...
# (editar archivos)

# Al terminar
git status
git add .
git commit -m "Descripción de cambios"
git push
```

### Clonar proyecto existente

```bash
git clone https://github.com/usuario/repo.git
cd repo
git checkout nombre-rama
```

---

## 🔧 COMANDOS DE UTILIDAD

```bash
git --version
# Muestra la versión de Git instalada

code .
# Abre la carpeta actual en VSCode (desde Git Bash)

git branch --set-upstream-to=origin/rama
# Vincula tu rama local con una rama remota

rm -rf .git
# Elimina Git del proyecto (¡cuidado! borra TODO el historial)
```

---

## 🚨 SOLUCIÓN DE PROBLEMAS COMUNES

### Error: "not a git repository"
```bash
git init
# Solución: Inicializa Git en la carpeta
```

### Error: "remote origin already exists"
```bash
git remote remove origin
git remote add origin https://github.com/usuario/repo.git
# Solución: Elimina y vuelve a agregar el remoto
```

### Error: "refusing to delete the current branch"
```bash
# Solución: Cambia la rama por defecto en GitHub Settings
# Luego: git push origin --delete nombre-rama
```

### Error: "src refspec does not match any"
```bash
git add .
git commit -m "Initial commit"
git push -u origin main
# Solución: Necesitas hacer un commit primero
```

### Error: "Could not resolve host: github.com"
```bash
# Solución: Problema de conexión a internet
# Verifica tu red o espera un momento
```

---

## 📋 FLUJOS ESPECÍFICOS

### Renombrar rama (local + remoto)
```bash
git branch -m nuevo-nombre          # Renombrar local
git push -u origin nuevo-nombre     # Subir con nuevo nombre
# (Cambiar default branch en GitHub si es necesario)
git push origin --delete viejo-nombre  # Eliminar rama vieja
```

### Sincronizar con rama remota existente
```bash
git fetch                           # Actualizar info
git checkout nombre-rama            # Cambiar a la rama
git pull origin nombre-rama         # Descargar últimos cambios
```

### Conectar proyecto local con GitHub
```bash
git remote add origin https://github.com/usuario/repo.git
git push -u origin main
```

---

## 💡 TIPS IMPORTANTES

1. **Siempre haz `git pull` antes de trabajar** (para tener los últimos cambios)
2. **Haz commits frecuentes** con mensajes descriptivos
3. **Nunca borres la carpeta `.git`** (contiene todo el historial)
4. **Usa `git status`** constantemente para saber dónde estás
5. **La rama por defecto** ya no se llama "master", ahora es "main"
6. **Después del primer `push -u`**, solo necesitas `git push`

---

## 🎯 COMANDOS MÁS USADOS DEL DÍA A DÍA

```bash
git status          # Ver estado
git pull            # Descargar cambios
git add .           # Preparar archivos
git commit -m "..." # Guardar cambios
git push            # Subir cambios
git branch          # Ver ramas
git checkout rama   # Cambiar de rama
```

---

## 📖 GLOSARIO RÁPIDO

- **Repository (Repo):** Carpeta con control de versiones
- **Commit:** Guardar cambios en el historial
- **Branch (Rama):** Línea independiente de desarrollo
- **Remote (Remoto):** Repositorio en internet (GitHub)
- **Origin:** Nombre por defecto del repositorio remoto
- **Pull:** Descargar cambios del remoto
- **Push:** Subir cambios al remoto
- **Fetch:** Actualizar info sin fusionar
- **Clone:** Copiar un repositorio remoto a local
- **Staging:** Área de preparación antes del commit
- **HEAD:** Apuntador a tu posición actual
