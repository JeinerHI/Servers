> [!note]
>Ventaja de linux es que openssl ya viene instalado por defecto, así que sigamos

## Configuración de la certificadora

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

### 2. ==Procedo a generar mi certificación raíz==

#### A. Conceptos clave!!
##### 🥇 Certificadora Raíz (Root CA)

El certificado raíz (`certs/ca.cert.pem`) es el **Root CA**.

- **Identificación:** Es el certificado **autofirmado** (se firma a sí mismo con su propia clave privada).
    
- **Propósito:** Es el **punto de máxima confianza** en la jerarquía de PKI. La clave privada del Root CA (`private/ca.key.pem`) se usa para firmar _todos_ los certificados que vienen después. Por seguridad, la clave del Root CA debe **mantenerse fuera de línea** (offline) y usarse lo menos posible.

##### 🥈 Certificadora Intermedia (Intermediate CA)

La Certificadora Intermedia es un certificado que se crea _después_ del Root CA y es **firmado por el Root CA**.

- **Propósito:** Actúa como un delegado del Root CA. Todos los certificados de servidor (como el de `vsftpd`) y de usuario se firman con la clave privada de la CA Intermedia. Esto se hace para que, si la clave intermedia se ve comprometida, puedas revocarla sin tener que destruir y reemplazar toda la jerarquía de confianza del Root CA.
    
- **Sección en tu `openssl.cnf`:** La configuración de tu profesor incluye la sección `[ v3_intermediate_ca ]` precisamente para este propósito, pero **aún no la hemos usado**.

#### B. Generación de Root CA
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