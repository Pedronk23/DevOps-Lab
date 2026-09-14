# Ataques históricos a SSL/TLS

## BEAST attack

Para cifrar un bloque de datos, **CBC** (*Cipher Block Chaining*) necesita un "número
aleatorio inicial" llamado **vector de inicialización (IV)**.

- **El truco**: un atacante que pudiera "escuchar" el tráfico de red (*man in the middle*) y
  lograra meter un script malicioso podía **predecir esos IVs**.
- **El resultado**: enviando datos específicos y viendo cómo se descifraban usando esos IVs,
  el atacante descifraba la información bloque por bloque.

## Heartbleed bug

Existe en TLS la función **Heartbeat**: tu ordenador y el servidor mantienen la conexión
enviándose una palabra de, por ejemplo, 4 letras ("HOLA").

- **El truco**: el atacante decía al servidor: *"te envío una palabra de 5000 letras… por
  cierto, la palabra es 'A'"*.
- **El resultado**: el servidor, confundido pero obediente, leía la "A" y **rellenaba las 4999
  letras restantes con lo que hubiera almacenado en memoria RAM**. Eso incluía contraseñas,
  claves privadas, correos electrónicos, etc.

> Fue un fallo de implementación de OpenSSL (falta de validación de longitud), no del
> protocolo TLS en sí.

## POODLE attack

Atacaba directamente a **SSL 3.0**, un protocolo con 15 años y obsoleto, pero que los
servidores y navegadores seguían soportando por compatibilidad.

- **El truco**: el atacante, en medio de la conexión, interfería en el saludo inicial
  (*handshake*) entre navegador y servidor. Provocaba fallos para forzar un *downgrade* de
  TLS 1.2 a SSL 3.0.
- **El resultado**: una vez en SSL 3.0, atacaba cómo el protocolo manejaba el relleno
  (*padding*) y podía descifrar caracteres uno a uno.

## Lección común

Los tres se mitigan con lo mismo: **deshabilitar protocolos y cifrados antiguos** (SSL 3.0,
TLS 1.0/1.1, CBC con TLS antiguo), usar TLS 1.2+ con AEAD (AES-GCM, ChaCha20) y mantener
las librerías parcheadas.

## Resumen comparativo

| Ataque | Año | Dónde estaba el fallo | Qué se obtenía | Mitigación |
|---|---|---|---|---|
| **BEAST** | 2011 | CBC con IV predecible en TLS 1.0 | descifrar cookies bloque a bloque | TLS 1.1+, cifrados AEAD |
| **Heartbleed** | 2014 | **implementación** de OpenSSL (sin validar longitud) | 64 KB de RAM por petición: claves, contraseñas | parchear OpenSSL y **rotar claves** |
| **POODLE** | 2014 | padding de SSL 3.0 + downgrade forzado | descifrar byte a byte | desactivar SSL 3.0, `TLS_FALLBACK_SCSV` |
| **FREAK / Logjam** | 2015 | cifrados "export" de 512 bits heredados | romper la clave de sesión | eliminar cifrados export y DH débil |
| **CRIME / BREACH** | 2012-13 | compresión TLS/HTTP | deducir secretos por el tamaño | desactivar compresión TLS |
| **ROBOT** | 2017 | relleno RSA PKCS#1 v1.5 | descifrar sesiones | RSA estático fuera (TLS 1.3) |

## Patrón común

```
   1. Un protocolo o cifrado antiguo sigue habilitado "por compatibilidad"
                        │
   2. El atacante fuerza un downgrade o explota el modo débil
                        │
   3. Descifra poco a poco (byte a byte) o filtra memoria
                        │
   ➜  La defensa casi nunca es "más bits": es RETIRAR lo viejo
```

## Configuración segura hoy (nginx)

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;     # en TLS 1.3 manda el cliente
ssl_session_tickets off;
ssl_stapling on;
add_header Strict-Transport-Security "max-age=63072000" always;
```

## Cómo auditar tu propio servicio

```bash
# cifrados y versiones soportadas
nmap --script ssl-enum-ciphers -p 443 app.ejemplo.com

# comprobar que TLS 1.0 está cerrado (debe fallar)
openssl s_client -connect app.ejemplo.com:443 -tls1

# suite completa de pruebas
testssl.sh https://app.ejemplo.com
```

Servicios externos: SSL Labs (`ssllabs.com/ssltest`) para lo público; `testssl.sh` para lo
interno, que no sale de tu red.

## Lección para el día a día

- Mantener OpenSSL/librerías parcheadas: Heartbleed no fue un fallo de diseño, fue un bug.
- Tras una filtración de clave privada, **rotar el certificado y revocar el anterior**:
  parchear no basta.
- Desactivar por defecto y habilitar solo lo necesario, en vez de lo contrario.
- Monitorizar la configuración TLS igual que monitorizas la caducidad del certificado.
