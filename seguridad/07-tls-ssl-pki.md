# TLS, SSL y PKI

## SSL y TLS

**SSL** (*Secure Sockets Layer*): crear un canal seguro entre tu navegador y el servidor web.
**Obsoleto** (ataque POODLE).

**TLS** (*Transport Layer Security*): sucesor de SSL.

| Versión | Estado |
|---|---|
| TLS 1.0 | 1999, el IETF estandarizó el protocolo; en esencia era "SSL 3.1". Obsoleto |
| TLS 1.1 | obsoleto |
| TLS 1.2 | lanzado en 2008, muy utilizado, pero retirándose poco a poco |
| TLS 1.3 | lanzado en 2018, estándar actual, más rápido y seguro |

## Cómo funciona una web segura (handshake)

1. **Saludo**: el navegador le dice al servidor: "hola, quiero conectar, estas son las
   versiones de TLS que entiendo".
2. **Identificación**: el servidor responde con su **certificado digital** (que incluye su
   clave pública). Demuestra a la web ser quien es.
3. **Clave secreta**: navegador y servidor generan una clave de cifrado **única y simétrica**
   para esa sesión.
4. **Canal seguro**: desde ese milisegundo, toda la información viaja encriptada. Para un
   tercero solo son símbolos aleatorios.

## PKI (Public Key Infrastructure)

Conjunto de leyes, reglas, personas y servidores que se necesita para crear, gestionar,
distribuir y revocar certificados digitales.

Para que TLS funcione, necesita una **clave pública** (que todo el mundo ve) y una **privada**
(que solo tú tienes). La PKI es la estructura que asegura que esa clave pública realmente te
pertenece a ti y no a un suplantador.

## El handshake, paso a paso

```
   NAVEGADOR                                        SERVIDOR
      │── ClientHello ─────────────────────────────────►│
      │   versiones TLS, cipher suites, SNI, random     │
      │                                                 │
      │◄── ServerHello + Certificate ───────────────────│
      │    elige cifrado, envía su cadena de certs      │
      │                                                 │
      │── verifica el certificado ──┐                   │
      │   · ¿firmado por una CA de confianza?           │
      │   · ¿el nombre coincide (SAN)?                  │
      │   · ¿está vigente? ¿revocado (OCSP)?            │
      │                                                 │
      │◄── intercambio de claves (ECDHE) ──────────────►│
      │    ambos derivan la MISMA clave de sesión       │
      │                                                 │
      │── Finished ────────────────────────────────────►│
      │◄─────────── canal cifrado con AES ─────────────►│
```

En **TLS 1.3** el handshake se reduce a 1 ida y vuelta (1-RTT, o 0-RTT con reanudación), y se
eliminan los cifrados antiguos: no hay RSA para intercambio de claves, ni CBC, ni RC4.

## Asimétrico vs simétrico: por qué se usan los dos

```
   ┌── ASIMÉTRICO (RSA / ECDSA) ──┐   ┌── SIMÉTRICO (AES) ─────────┐
   │ lento                        │   │ rapidísimo                 │
   │ resuelve: identidad y        │   │ resuelve: cifrar el tráfico │
   │ acordar una clave secreta    │   │ pero necesita clave común  │
   └──────────────┬───────────────┘   └──────────────▲─────────────┘
                  └── se usa solo al principio ──────┘
                      para acordar la clave de sesión
```

## Forward secrecy

Con ECDHE, la clave de sesión **no se puede recuperar** ni teniendo la clave privada del
servidor después: cada sesión usa claves efímeras. Por eso un robo futuro de la clave privada
no descifra tráfico grabado en el pasado.

## Anatomía de un certificado X.509

```
   ┌──────────────────────────────────────────────┐
   │ Subject      CN=app.ejemplo.com             │
   │ SAN          app.ejemplo.com, www.ejemplo.com│ ← lo que validan los navegadores
   │ Issuer       CN=R3, O=Let's Encrypt         │
   │ Validez      notBefore / notAfter           │
   │ Clave pública (RSA 2048 / EC P-256)         │
   │ Extensiones  keyUsage, extendedKeyUsage     │
   │ Firma de la CA                              │
   └──────────────────────────────────────────────┘
```

> Hoy el **CN se ignora**: si el dominio no está en el SAN, el navegador da error.

## Cadena de confianza

```
   Root CA (en el almacén del sistema/navegador, autofirmada)
      └── Intermediate CA (la que firma de verdad)
             └── Certificado de tu servidor (leaf)
```

El servidor debe enviar **leaf + intermedios** (fullchain), no solo el suyo. Error clásico:
funciona en el navegador (que cachea intermedios) y falla en `curl` o en Java.

## Formatos de fichero

| Extensión | Contenido |
|---|---|
| `.pem` / `.crt` / `.cer` | base64 con cabeceras `-----BEGIN CERTIFICATE-----` |
| `.key` | clave privada (PEM) |
| `.der` | igual que PEM pero en binario |
| `.p12` / `.pfx` | contenedor con certificado + clave + cadena, protegido por contraseña (típico en Windows y Java) |
| `.csr` | petición de firma que envías a la CA |

## Comandos de OpenSSL que se usan a diario

```bash
# ver un certificado local
openssl x509 -in cert.pem -noout -text
openssl x509 -in cert.pem -noout -subject -issuer -dates

# inspeccionar un servidor en producción
openssl s_client -connect app.ejemplo.com:443 -servername app.ejemplo.com </dev/null
echo | openssl s_client -connect app.ejemplo.com:443 2>/dev/null | openssl x509 -noout -dates

# generar clave + CSR
openssl req -new -newkey rsa:2048 -nodes -keyout app.key -out app.csr \
  -subj "/CN=app.ejemplo.com"

# comprobar que una clave y un certificado son pareja (los módulos deben coincidir)
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa  -noout -modulus -in app.key  | openssl md5

# convertir a PKCS#12
openssl pkcs12 -export -out app.pfx -inkey app.key -in cert.pem -certfile chain.pem
```

## Errores típicos y su causa

| Error del navegador / cliente | Causa |
|---|---|
| `NET::ERR_CERT_AUTHORITY_INVALID` | CA no confiable (autofirmado o falta el intermedio) |
| `NET::ERR_CERT_COMMON_NAME_INVALID` | el dominio no está en el SAN |
| `NET::ERR_CERT_DATE_INVALID` | caducado, o el reloj del cliente está mal |
| `unable to get local issuer certificate` (curl) | falta la cadena de intermedios en el servidor |
| `handshake failure` | no hay cipher suite ni versión TLS en común |

## El handshake, con sus mensajes reales

```
  CLIENTE                                          SERVIDOR
     │                                                 │
     │── ClientHello ─────────────────────────────────►│
     │   versiones TLS, cipher suites, SNI, random     │
     │                                                 │
     │◄─ ServerHello ──────────────────────────────────│
     │   versión y cipher elegidos, random             │
     │◄─ Certificate (+ cadena intermedia) ────────────│
     │◄─ ServerKeyExchange / Finished ─────────────────│
     │                                                 │
     │   valida: firma de la CA, fecha, CN/SAN, revocación
     │                                                 │
     │── KeyExchange (ECDHE) ─────────────────────────►│
     │══ clave de sesión simétrica derivada ═══════════│
     │── Finished ────────────────────────────────────►│
     │◄─ Finished ─────────────────────────────────────│
     │══════ datos de aplicación cifrados ═════════════│
```

En **TLS 1.3** el handshake se reduce a **1 RTT** (o 0 con reanudación) y se eliminaron los
cifrados antiguos (RSA estático, CBC, RC4, compresión), que eran la raíz de la mayoría de
ataques.

## Cadena de confianza

```
   ┌──────────────────────┐
   │  Root CA             │  autofirmada, está en el root store del SO/navegador
   │  (offline, 10-20 a.) │
   └──────────┬───────────┘
              │ firma
   ┌──────────▼───────────┐
   │  Intermediate CA     │  la que firma el día a día
   └──────────┬───────────┘
              │ firma
   ┌──────────▼───────────┐
   │  Certificado de tu   │  app.ejemplo.com (90 días con Let's Encrypt)
   │  servidor (leaf)     │
   └──────────────────────┘
```

> Error clasiquísimo: servir solo el certificado hoja sin la intermedia. En tu navegador
> funciona (la tiene cacheada) y en `curl` o en un móvil falla. Hay que servir la
> **cadena completa** (`fullchain.pem`).

## Qué hay dentro de un certificado

| Campo | Contenido |
|---|---|
| `Subject` / CN | a quién identifica |
| `SAN` (Subject Alternative Name) | los dominios válidos: **es el campo que se valida hoy**, el CN es histórico |
| `Issuer` | quién lo firmó |
| `Not Before` / `Not After` | validez |
| Clave pública | la del servidor |
| `Key Usage` / `EKU` | para qué se puede usar (servidor web, firma de código…) |
| Firma | de la CA emisora |

## Formatos de fichero

```
   PEM  (.pem .crt .cer .key)  → texto base64 con -----BEGIN ...-----   ← el habitual en Linux
   DER  (.der .cer)            → binario                                ← Java, Windows
   PKCS#12 (.p12 .pfx)         → certificado + clave privada en un fichero, con contraseña
   JKS                         → almacén de Java (keytool)
```

```bash
# convertir PEM → PKCS12 (típico para Tomcat o Windows)
openssl pkcs12 -export -in fullchain.pem -inkey privkey.pem -out cert.pfx
```

## Comandos de diagnóstico

```bash
# ver el certificado que sirve un host
openssl s_client -connect app.ejemplo.com:443 -servername app.ejemplo.com </dev/null \
  | openssl x509 -noout -text

# solo fechas y sujeto
echo | openssl s_client -connect app.ejemplo.com:443 2>/dev/null \
  | openssl x509 -noout -subject -dates

# inspeccionar ficheros locales
openssl x509 -in cert.pem -noout -text
openssl rsa -in privkey.pem -check
openssl req -in peticion.csr -noout -text

# ¿coinciden certificado y clave privada?
openssl x509 -noout -modulus -in cert.pem  | openssl md5
openssl rsa  -noout -modulus -in privkey.pem | openssl md5   # deben dar lo mismo

# ver versiones y cifrados soportados
nmap --script ssl-enum-ciphers -p 443 app.ejemplo.com
```

## Errores TLS y su causa

| Error del navegador/cliente | Causa |
|---|---|
| `NET::ERR_CERT_AUTHORITY_INVALID` | CA no confiable (autofirmado o falta la intermedia) |
| `NET::ERR_CERT_COMMON_NAME_INVALID` | el dominio no está en el SAN |
| `NET::ERR_CERT_DATE_INVALID` | caducado, o reloj del cliente mal |
| `unable to get local issuer certificate` | falta la cadena en el servidor |
| `handshake failure` | no hay cifrados o versiones en común |
