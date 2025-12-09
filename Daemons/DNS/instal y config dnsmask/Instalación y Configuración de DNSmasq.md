Paso 1: Instalar DNSmasq
```bash
sudo apt update
sudo apt install dnsmasq -y
```
-- Verificar instalación
```bash
dnsmasq --version
```
![[Pasted image 20251022155337.png]]

Paso 2: genero copia del archivo de configuración para tener opción de back up.
```bash
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.bak
sudo nano /etc/dnsmasq.conf
```
Modificaciones sobre el archivo:

![[Pasted image 20251022161833.png]]

Luego de modificar, reinicio el DNSmasq para aplicar la configuración con el siguiente comando
```bash
sudo systemctl restart dnsmasq
```

Después de este intento he tenido un error que me indica la terminal es debido a que el puerto 53 no está disponible para su uso, frente a lo cual ejecuto el siguiente comando para ver qué proceso tiene ocupado el puerto:
```bash
sudo lsof -i :53
```
 Y luego tengo el siguiente resultado:
![[Pasted image 20251022162715.png]]
Frente a esto, procedo a detener y deshabilitar temporalmente systemd-resolved
```bash 
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```

Luego de esto paso a reconfigurar /etc/resolv.conf después de guardar una copia de seguridad
```bash
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
```
Ahora intento reiniciar el dnsmasq así:
```bash
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
```

## ... Redireccionamiento después de varios intentos
Después de realizar muchos intentos fallidos por vaciar el puerto 53 para poder levantar ahí el dnsmasq, decidí cambiar el puerto del que escuhará el servidor, quedando mi archivo de configuración de la siguiente manera:
```bash
sudo nano /etc/dnsmasq.conf
```
Luego agrego las líneas de configuración:
```conf
# =======================
# Archivo: /etc/dnsmasq.conf
# =======================

# Usa el puerto 5353 en lugar del 53
port=5353

# Escucha en localhost
listen-address=127.0.0.1
listen-address=172.22.143.8

# Interfaces donde escuchar
interface=lo
interface=eth0

# No leas /etc/resolv.conf para servidores upstream
no-resolv

# Dominio local que se gestionará por este DNS
domain=dominio-daw.com

# Servidor DNS externo para consultas no locales (reenviador)
server=8.8.8.8
server=8.8.4.4

# Nombre de host local
# Formato: address=/nombre_dominio/IP
address=/servidor.dominio-daw.com/172.22.143.8

# Archivo de hosts local (aquí defino mis hosts)
addn-hosts=/etc/dnsmasq.hosts

# Registrar los nombres de los equipos DHCP en el DNS local
expand-hosts

```

Ahora procedo a reinicar dnsmasq.
```bash
sudo systemctl restart dnsmasq
```
Sin embargo tengo el siguiente error:
![[Pasted image 20251028145229.png]]

### Solución de error
	Empiezo por ver los logs del error
```bash
	sudo journalctl -xeu dnsmasq.service
```
![[Pasted image 20251028145552.png]]
Con esto consigo identificar que el error sigue siendo que dnsmasq intenta crear el escuchador en el puerto 53, así que paso a ver porqué está ignorando mi archivo de configuración y lo corrijo.

	Después de ejecutar el comando `cat /etc/dnsmasq.conf` identifico que este es un archivo diferente, que he creado antes en mis otros intentos de configuración

Paso a ver que archivos con el nombre dnsmasq tengo en el directorio etc
```bash
ls -la /etc/dnsmasq* 2>/dev/null
```
![[Pasted image 20251028150657.png]]

Descubro que el archivo de configuración que está funcionando es el "dnsmasq.conf" así que paso a configurar este correctamente con los siguientes parámetros:
```conf
# =======================
# Archivo: /etc/dnsmasq.conf
# =======================

# Usa el puerto 5353 en lugar del 53
port=5353

# Escucha en localhost
listen-address=127.0.0.1
listen-address=172.22.143.8

# Interfaces donde escuchar
interface=lo
interface=eth0

# No leas /etc/resolv.conf para servidores upstream
no-resolv

# Dominio local que se gestionará por este DNS
domain=dominio-daw.com

# Servidor DNS externo para consultas no locales (reenviador)
server=8.8.8.8
server=8.8.4.4

# Nombre de host local
# Formato: address=/nombre_dominio/IP
address=/servidor.dominio-daw.com/172.22.143.8

# Archivo de hosts local (aquí definirás tus hosts)
addn-hosts=/etc/dnsmasq.hosts

# Registrar los nombres de los equipos DHCP en el DNS local
expand-hosts

```

## Pruebas de Funcionamiento

- Reinicio del DNSmasq para aplicar cambios:
```bash
sudo systemctl restart dnsmasq
```

- Verificar el estado del servicio
```bash
sudo systemctl status dnsmasq
```
	Logro evidenciar que el sistema está corriendo bien

![[Pasted image 20251028153359.png]]

#### - Pruebas de resolución de DNS
![[Pasted image 20251029111827.png]]
![[Pasted image 20251029111844.png]]
![[Pasted image 20251029111858.png]]
