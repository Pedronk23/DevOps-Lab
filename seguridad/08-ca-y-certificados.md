# Certificados: CA, OCSP, PKP y SCEP

## CA (Certificate Authority)

Empresas del sector: **Let's Encrypt** (gratuita y automatizable), DigiCert, Sectigo.

Todo el mundo puede crear una **CA privada** o una **CA autofirmada**.

> Problema: en el navegador la web saldrá como **NO SEGURA**. Google, Apple y Microsoft tienen
> una lista interna de "CAs de confianza" (llamada *root store*), y una CA propia no está ahí
> salvo que la instales tú en los dispositivos.

### CA públicas

- Para páginas web, tiendas online, blogs, etc.
- 100 % gratis con Let's Encrypt.

### CA privadas

- Redes internas, entornos de desarrollo, laboratorios caseros.
- Tú creas e instalas tu CA en los dispositivos de tu casa o empresa.

## Protocolos relacionados

| Siglas | Qué es |
|---|---|
| **OCSP** (Online Certificate Status Protocol) | consultar si un certificado es válido o está caducado/revocado |
| **PKP** (Public Key Pinning) | en desuso. Objetivo original: detectar si una CA estaba comprometida o hackeada |
| **SCEP** (Simple Certificate Enrollment Protocol) | automatizar la entrega de certificados a miles de dispositivos de la red (p. ej. generar 5000 certificados, uno para cada persona de una gran empresa) |

## Crear una CA

Se pueden crear CAs en Windows, macOS o Linux con **OpenSSL**.

En Kubernetes, lo equivalente automatizado es **cert-manager** (ver
[cert-manager](../kubernetes/07-ingress-controllers-y-cert-manager.md)).

```bash
# clave privada de la CA
openssl genrsa -out ca.key 4096
# certificado raíz autofirmado
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt
```

## Ciclo de vida de un certificado

```
   1. generar clave privada          openssl genrsa / genpkey
            │
   2. crear CSR (petición)           openssl req -new
      contiene: clave pública + dominios + datos
            │
   3. la CA valida y firma           → certificado
      · pública: reto HTTP-01 / DNS-01
      · privada: el admin lo firma
            │
   4. instalar en el servidor        cert + clave + cadena
            │
   5. renovar antes de caducar       cron / certbot / cert-manager
            │
   6. revocar si se filtra la clave  CRL / OCSP
```

## Crear una CA privada con OpenSSL (laboratorio)

```bash
# 1. clave y certificado raíz de la CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/C=ES/O=Lab/CN=Lab Root CA" -out ca.crt

# 2. clave y CSR del servidor
openssl genrsa -out servidor.key 2048
openssl req -new -key servidor.key -subj "/CN=app.lab.local" -out servidor.csr

# 3. extensiones con SAN (imprescindible hoy)
cat > san.cnf <<'EOC'
subjectAltName = DNS:app.lab.local, DNS:*.lab.local, IP:192.168.1.50
EOC

# 4. firmar con la CA
openssl x509 -req -in servidor.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out servidor.crt -days 365 -sha256 -extfile san.cnf

# 5. verificar
openssl verify -CAfile ca.crt servidor.crt
```

### Instalar la CA como confiable

```bash
# Debian/Ubuntu
sudo cp ca.crt /usr/local/share/ca-certificates/lab-root.crt
sudo update-ca-certificates

# RHEL/Rocky
sudo cp ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

```powershell
# Windows (como administrador)
Import-Certificate -FilePath .\ca.crt -CertStoreLocation Cert:\LocalMachine\Root
```

## Let's Encrypt: los dos retos

```
   HTTP-01                                DNS-01
   ─────────────────────────              ─────────────────────────
   la CA pide:                            la CA pide:
   http://dominio/.well-known/            registro TXT
     acme-challenge/<token>                 _acme-challenge.dominio
            │                                      │
   necesita el puerto 80 abierto          no necesita puertos abiertos
   no sirve para wildcards                SÍ sirve para *.dominio
```

```bash
# certificado normal
sudo certbot --nginx -d app.ejemplo.com

# wildcard por DNS
sudo certbot certonly --manual --preferred-challenges dns -d '*.ejemplo.com'

# renovación (certbot instala un timer automáticamente)
sudo certbot renew --dry-run
```

Los certificados de Let's Encrypt duran **90 días**: la renovación automática no es opcional,
es parte del diseño.

## Automatización en cada entorno

| Entorno | Herramienta |
|---|---|
| Servidor Linux suelto | `certbot` + timer de systemd |
| Kubernetes | `cert-manager` (recurso `Certificate`) |
| Windows / IIS | `win-acme`, o ADCS con autoinscripción |
| Miles de dispositivos | **SCEP** / EST |
| Certificados internos con Vault/OpenBao | motor de secretos PKI, emisión con TTL corto |

## Revocación: CRL vs OCSP

```
   CRL                                OCSP
   ─────────────────────              ───────────────────────────
   lista completa de revocados        pregunta por UN certificado
   se descarga entera                 respuesta pequeña y rápida
   puede pesar megas                  problema: fuga de privacidad
                                      solución: OCSP stapling (lo
                                      sirve el propio servidor web)
```

```nginx
# OCSP stapling en nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/ssl/chain.pem;
```

## Buenas prácticas

- Claves privadas con permisos `600` y nunca en el repositorio de código.
- SAN siempre; el CN por sí solo ya no lo valida ningún navegador.
- Vigilar la caducidad con monitorización (Prometheus `blackbox_exporter` o un check simple).
- TTL cortos + automatización es más seguro que certificados de 2 años renovados a mano.
- Root CA offline; firmar siempre con una intermedia.
