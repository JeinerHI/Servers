> [!note]
>Ventaja de linux es que openssl ya viene instalado por defecto, así que sigamos

## Configuración de la certificadora
### Conceptos clave!!
#### 🥇 Certificadora Raíz (Root CA)

El certificado raíz (`certs/ca.cert.pem`) es el **Root CA**.

- **Identificación:** Es el certificado **autofirmado** (se firma a sí mismo con su propia clave privada).
    
- **Propósito:** Es el **punto de máxima confianza** en la jerarquía de PKI. La clave privada del Root CA (`private/ca.key.pem`) se usa para firmar _todos_ los certificados que vienen después. Por seguridad, la clave del Root CA debe **mantenerse fuera de línea** (offline) y usarse lo menos posible.

#### 🥈 Certificadora Intermedia (Intermediate CA)

La Certificadora Intermedia es un certificado que se crea _después_ del Root CA y es **firmado por el Root CA**.

- **Propósito:** Actúa como un delegado del Root CA. Todos los certificados de servidor (como el de `vsftpd`) y de usuario se firman con la clave privada de la CA Intermedia. Esto se hace para que, si la clave intermedia se ve comprometida, puedas revocarla sin tener que destruir y reemplazar toda la jerarquía de confianza del Root CA.
    
- **Sección en tu `openssl.cnf`:** La configuración de tu profesor incluye la sección `[ v3_intermediate_ca ]` precisamente para este propósito, pero **aún no la hemos usado**.


### 1. ==Primer paso es preparar el entorno de la CA raíz==

#### A. Crear la estructura de Carpetas y archivos en la home con comandos `mkdir` para directorios y `touch` para archivos
Creo las carpetas críticas y ficheros que la configuración de OpenSSL (en `openssl.cnf`) esperará:

>[!note]
>Los ficheros `serial` y `crlnumber` lo creo inicializado a 1000 para arrancar el consecutivo que incrementará automáticamente, es una buena práctica, así:
>```bash
>echo 1000 > serial
>echo 1000 > crlnumber
>```

```ini
~/ca/root-ca/
├── certs/ # Donde se almacenan los certificados de la CA. 
├── crl/ # Nuevo: Donde se almacenarán las Listas de Revocación (CRL). 
├── newcerts/ # Donde se almacenan los certificados firmados. 
├── private/ # Donde se almacena la clave privada de la CA (ca.key.pem). 
├── index.txt # Base de datos de certificados emitidos. 
├── serial # Número de serie del próximo nuevo certificado.
├── crlnumber # Número de secuencia (o número CRL) de la próxima Lista de Revocación de Certificados (CRL) que la CA va a generar.
├── openssl.cnf # Archivo de configuración que usaré.
```

#### B. Editar el archivo de configuración de la CA, que he nombrado openssl.cnf

Este paso implica modificar el archivo **`openssl.cnf`** que he creado y actúa como las **"reglas"** de la Autoridad Certificadora. No es un comando, sino un **archivo de texto** con secciones (como `[ ca ]`, `[ CA_default ]`, `[ req ]`) que definen:

- **Ubicaciones** (`$dir/certs`, `$dir/index.txt`, etc.): Le dice a OpenSSL dónde encontrar todos los archivos de la CA.
    
- **Políticas** (`policy_strict`): Define qué campos (país, organización, etc.) deben coincidir o son requeridos al firmar un certificado.
	
- **Restricciones de la CA** (`basicConstraints = critical, CA:true`): Le dice a cualquier sistema que confíe en este certificado que **sí**, tiene permiso para firmar otros certificados (es una CA).

***Configuracion de `openssl.cnf`***
```bash
touch ca.cnf
sudo nano ca.cnf
```

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir               = /homw/dawoodlinux/ca/root-ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30
default_md        = sha256
name_opt          = ca_default
cert_opt          = ca_default
default_days      = 3650
preserve          = no
policy            = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 4096
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca

[ req_distinguished_name ]
countryName                     = País (código de 2 letras)
stateOrProvinceName             = Estado o Provincia
localityName                    = Ciudad
0.organizationName              = Organización
organizationalUnitName          = Unidad Organizacional
commonName                      = Nombre Común
emailAddress                    = Correo Electrónico

countryName_default             = ES
stateOrProvinceName_default     = Madrid
localityName_default            = Madrid
0.organizationName_default      = Mi Empresa CA
organizationalUnitName_default  = Departamento TI
emailAddress_default            = ca@miempresa.com

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ usr_cert ]
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "Certificado de Cliente OpenSSL"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "Certificado de Servidor OpenSSL"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost
DNS.2 = *.miempresa.com
IP.1 = 127.0.0.1

[ crl_ext ]
authorityKeyIdentifier=keyid:always

[ ocsp ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

### 2. ==Generación del certificado raíz (Root CA)==
Desde el directorio base de la CA (`~/ca/root-ca`). Ejecuto el siguiente comando para generar la clave privada de 4096 bits y el certificado Root CA autofirmado:

>[!note]
	>Al ejecutar, lo primero que me pedirá es una frase segura, que básicamente es una frase que servirá como segundo autentificador de seguridad antes de la contraseña ya oculta. en mi caso puse "frasesegura" para el ejemplo

```bash
openssl req -new -x509 -days 3650 -extensions v3_ca -config openssl.cnf -keyout private/ca.key.pem -out certs/ca.cert.pem
```

| **Opción**                       | **Función**           | **Explicación**                                                                                                                                                         |
| -------------------------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`-new -x509`**                 | Certificado Root      | Crea una nueva clave y un certificado **autofirmado** (el certificado Root).                                                                                            |
| **`-days 3650`**                 | Validez               | El certificado será válido por **10 años** (3650 días), tal como está definido en tu archivo de configuración.                                                          |
| **`-extensions v3_ca`**          | Extensiones           | Fuerza el uso de la sección **`[ v3_ca ]`** del archivo `openssl.cnf`, asegurando que el certificado se marque correctamente como una **Autoridad Certificadora (CA)**. |
| **`-config openssl.cnf`**        | Configuración         | Indica a OpenSSL que use las reglas avanzadas que has configurado en tu archivo `openssl.cnf`.                                                                          |
| **`-keyout private/ca.key.pem`** | Salida de Clave       | Guarda la **clave privada** de la CA en el directorio `private/`. Se te pedirá que ingreses una **contraseña (`passphrase`)** para proteger esta clave.                 |
| **`-out certs/ca.cert.pem`**     | Salida de Certificado | Guarda el **certificado público** de la CA en el directorio `certs/`. Este es el archivo que se usará para **establecer la confianza** en todos los sistemas.           |
>[!note]
>Al rellenar los campos que me pide al final, ya tendré el certificado raíz y la contraseña privada del mismo hechas, puedo pasar a generar los demás certificados que siempre deberán ser firmados por una certificadora, inicialmente la que acabo de crear, y si con esta certifico a una certificadora intermedia, entonces la intermedia firmará los demás certificados ya que ha sido autorizada por la raíz.

### 3. ==Preparación entorno de Certificadora Intermedia CA Intermedia==
**¿Por qué una intermedia?** Por seguridad. Si la clave privada de la CA Raíz se ve comprometida, toda la confianza se rompe. Al usar una intermedia, se mantiene la Raíz desconectada (offline) y segura, y se usa la Intermedia para firmar los certificados del día a día.

Estos son los pasos para configurar la **CA Intermedia** usando OpenSSL.
#### A. Contenido del archivo openssl.cnf para la CA Intermedia
```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
# OJO: Aquí estoy respetando tu ruta relativa "./ca/intermediate-ca"
dir             = /home/dawoodlinux/ca/intermediate-ca
certs           = $dir/certs
crl_dir         = $dir/crl
new_certs_dir   = $dir/newcerts
database        = $dir/index.txt
serial          = $dir/serial
RANDFILE        = $dir/private/.rand
private_key     = $dir/private/intermediate.key.pem
certificate     = $dir/certs/intermediate.cert.pem
crlnumber       = $dir/crlnumber
crl             = $dir/crl/intermediate.crl.pem
crl_extensions  = crl_ext
default_crl_days= 30
default_md      = sha256
name_opt        = ca_default
cert_opt        = ca_default
default_days    = 375
preserve        = no
policy          = policy_loose

[ policy_strict ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ policy_loose ]
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
default_md          = sha256
x509_extensions     = v3_ca

[ req_distinguished_name ]
countryName                     = País (código de 2 letras)
stateOrProvinceName             = Estado o Provincia
localityName                    = Ciudad
0.organizationName              = Organización
organizationalUnitName          = Unidad Organizacional
commonName                      = Nombre Común
emailAddress                    = Correo Electrónico
countryName_default             = ES
stateOrProvinceName_default     = Madrid
localityName_default            = Madrid
0.organizationName_default      = Mi Empresa CA
organizationalUnitName_default  = Departamento TI
emailAddress_default            = ca@miempresa.com

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
# pathlen:0 asegura que esta CA no pueda crear otras CAs debajo de ella
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ usr_cert ]
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "Certificado de Cliente OpenSSL"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "Certificado de Servidor OpenSSL"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
# He comentado esto para que no afecte a la creación de la propia Intermedia

[ alt_names ]
DNS.1 = localhost
DNS.2 = servidor.miempresa.com
DNS.3 = *.miempresa.com
IP.1 = 127.0.0.1
IP.2 = 192.168.1.100

[ crl_ext ]
authorityKeyIdentifier=keyid:always

[ ocsp ]
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
```

#### B. Genero la estructura de directorios
```bash
# Crear directorio para la intermedia
mkdir /home/dawoodlinux/ca/intermediate-ca

# Crear subdirectorios de organización
cd /home/dawoodlinux/ca/intermediate-ca
mkdir certs crl csr newcerts private
chmod 700 private

# Crear archivos de control
touch index.txt
touch openssl_intermediate.cnf (haré un nano y pasaré la config del punto anterior)
echo 1000 > serial
echo 1000 > crlnumber
```

Tendré entonces la siguiente estructura:
```ini
~/ca/intermediate-ca/
├── certs/ # Almacenará los certificados de la CA Intermedia (intermediate.cert.pem).
├── crl/ # Almacenará las Listas de Revocación (CRL).
├── csr/ # Almacena las Solicitudes de Firma(CSR) para certificados de servidor|cliente.
├── newcerts/ # Almacena los certificados de servidor|cliente firmados por inter CA
├── private/ # Almacena la clave privada de la CA Intermedia (intermediate.key.pem). ¡Permisos 700!
├── index.txt # Base de datos de certificados emitidos por la CA Intermedia.
├── serial # Número de serie del próximo nuevo certificado a emitir.
├── crlnumber # Número de secuencia (o número CRL) de la próxima Lista de Revocación
└── openssl_intermediate.cnf # Archivo de configuración específico para la CA Intermedia. (Lo llamamos así para diferenciarlo de la Root)
```

### 4. ==Generación del certificado de la CA Intermedia==
Debo estar situado en el directorio principal del proyecto (ej. `~/ca`) y la configuración y directorios siguen la estructura definida (`./ca/intermediate-ca/`).
#### A. Generar la Clave Privada (Private Key)

Es crucial que esta clave esté protegida con una contraseña robusta y que tenga permisos restringidos. Usaremos 4096 bits y el cifrado AES-256.

>[!note]
>Se me pedirá nuevamente un pass phrase, usaré el mismo de la raíz para el ejemplo, pero añadiendo la letra "b" al final "frasesegurab"
```bash
# 1. Generar la clave privada de 4096 bits, encriptada con AES-256      
openssl genrsa -aes256 -out intermediate-ca/private/intermediate.key.pem 4096
```

#### B. Generar la solicitud de firma de certificado (CSR)

```bash
openssl req -config intermediate-ca/openssl_intermediate.cnf \
  -new -sha256 \
  -key intermediate-ca/private/intermediate.key.pem \
  -out intermediate-ca/csr/intermediate.csr.pem
```

#### C. Firma del (CSR) de la intermedia con la Root CA

```bash
openssl ca -config root-ca/openssl.cnf \
  -extensions v3_intermediate_ca \
  -days 3650 -notext -md sha256 \
  -in intermediate-ca/csr/intermediate.csr.pem \
  -out intermediate-ca/certs/intermediate.cert.pem
```

- Ver el certificado
```bash
openssl x509 -noout -text -in intermediate-ca/certs/intermediate.cert.pem
```
	
- Verificar la cadena de certificados
```bash
openssl verify -CAfile root-ca/certs/ca.cert.pem \
  intermediate-ca/certs/intermediate.cert.pem
```

#### D. Crear cadena de certificados
Explicación: La cadena de certificados contiene el certificado intermedio seguido del certificado raíz. Esta cadena permite a los clientes verificar toda la jerarquía de confianza. El orden es importante: primero el certificado más específico (intermedio) y luego el raíz.

```bash
cat intermediate-ca/certs/intermediate.cert.pem \
    root-ca/certs/ca.cert.pem > \
    intermediate-ca/certs/ca-chain.cert.pem

chmod 444 intermediate-ca/certs/ca-chain.cert.pem
```

### 5. ==Ejemplo de generar y firmar un certficado de  un servidor==

#### A. Si no la tengo, creo la estructura de directorio para los certificados del servidor
```bash
# Creo un directorio para los certificados de servidor (si no existe)
mkdir -p intermediate-ca/servers
cd intermediate-ca/servers/
mkdir private csr certs
```

#### B. Generación de clave del servidor
Explicación: Generamos una clave de 2048 bits (suficiente para servidores). Puedes usar 4096 bits para mayor seguridad.
Para uso en producción sin solicitar contraseña en cada reinicio (menos seguro pero práctico):

```bash
openssl genrsa -aes256 -out intermediate-ca/servers/private/servidor.miempresa.com.key.pem 2048
```

```bash
# Asegurar permisos (sólo lectura para el dueño)
chmod 400 intermediate-ca/servers/private/servidor.miempresa.com.key.pem
```

#### C. Modifico los nombres alternativos  en el openssl_intermediate.cnf
Edito el fragmento de alt_names con la siguiente configuración.
```ini
[ alt_names ]
DNS.1 = servidor.miempresa.com
DNS.2 = www.miempresa.com
DNS.3 = miempresa.com
DNS.4 = *.miempresa.com
IP.1 = 192.168.1.100
IP.2 = 10.0.0.50
```
	Explicación: Los navegadores modernos requieren que el nombre del servidor esté en SAN. Aquí defines todos los nombres DNS y direcciones IP que el certificado debe cubrir.
#### D. Creal la CSR del servidor
Ahora creamos la solicitud, asegurándonos de que el common name CN sea como lo especifiqué en los dns de la sección SANS en el archivo de configuración de la CA intermedia, para este caso sería `servidor.miempresa.com`

```bash
openssl req -config intermediate-ca/openssl_intermediate.cnf \
  -key intermediate-ca/servers/private/servidor.miempresa.com.key.pem \
  -new -sha256 \
  -out intermediate-ca/servers/csr/servidor.miempresa.com.csr.pem
```

#### E. Firmar la CSR con la CA intermedia

```bash
openssl ca -config intermediate-ca/openssl_intermediate.cnf -extensions server_cert \ -days 375 -notext -md sha256 \ -in intermediate-ca/servers/csr/servidor.miempresa.com.csr.pem \ 
-out intermediate-ca/servers/certs/servidor.miempresa.com.cert.pem
```

#### F. Verificar el certificado del servidor
>[!warning]
>Aqui no me resulta nada

```bash
# Ver el certificado
openssl x509 -noout -text \
  -in intermediate-ca/servers/certs/servidor.miempresa.com.cert.pem

# Verificar contra la cadena
openssl verify -CAfile intermediate-ca/certs/ca-chain.cert.pem \
  intermediate-ca/servers/certs/servidor.miempresa.com.cert.pem
```

Puntos a verificar:

- CN coincide con el nombre del servidor
- SAN contiene todos los nombres necesarios
- Extended Key Usage incluye TLS Web Server Authentication
- Issuer es la CA intermedia