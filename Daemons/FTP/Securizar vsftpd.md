Para asegurar mi servidor **vsftpd** con TLS/SSL, necesito crear un certificado que sea específico para este servidor y que esté **firmado por mi Root CA**. Se hace mediante los siguientes pasos:

## 1. Creo una solicitud de firma (CSR) para vsftpd

- Comando para generar la **clave privada** y la **Solicitud de Firma de Certificado (CSR)** para el servidor `vsftpd` (usando la plantilla `[ server_cert ]` de mi archivo ``openssl.cnf``):
```bash
openssl req -new -newkey rsa:4096 -nodes -keyout vsftpd.key.pem -out vsftpd.csr.pem -config openssl.cnf -extensions server_cert
```
>[!note]
>Me debo asegurar de que el **Nombre Común (CN)** sea el **nombre de dominio o IP** que usarán los clientes para conectarse al servidor FTP (p. ej., `ftp.misitio.com` o `172.30.76.144`).

## 2. Paso a firmar la CSR con el Root CA
Ahora uso la herramienta `openssl ca` para firmar la solicitud (`vsftpd.csr.pem`) con la clave de la CA. ==Importante estar seguro que ``openssl.cnf`` está en el directorio actual==
Me pedirá la frase segura ya que lo que estoy haciendo es certificando con la clave del Root CA

```bash
openssl ca -batch -config openssl.cnf -policy policy_strict -extensions server_cert -in vsftpd.csr.pem -out vsftpd.cert.pem
```