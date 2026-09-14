# SSH — MFA (Multi Factor Authentication)

El servidor te exige **dos o más factores diferentes** para comprobar tu identidad.

## Los tres factores

```
   ┌──────────────────┬──────────────────┬──────────────────┐
   │  ALGO QUE SABES  │  ALGO QUE TIENES │  ALGO QUE ERES   │
   ├──────────────────┼──────────────────┼──────────────────┤
   │  contraseña      │  clave privada   │  huella          │
   │  PIN             │  YubiKey         │  biometría       │
   │                  │  móvil con TOTP  │                  │
   └──────────────────┴──────────────────┴──────────────────┘
   MFA = combinar factores de columnas DISTINTAS
```

> Clave privada + passphrase **no** es MFA real: la passphrase protege la clave, pero ambos
> están en el mismo sitio (tu portátil). Clave + código TOTP del móvil sí lo es.

## Métodos

| Método | Detalle |
|---|---|
| **Public Key + Password** | combinación clásica sin usar herramientas externas: clave privada (en el PC) y luego contraseña tradicional |
| **Google Authenticator** (software MFA) | códigos temporales de un solo uso; te pide un código de seis dígitos |
| **YubiKey** (hardware MFA) | el método más seguro. Llave física USB: hay que tocar físicamente el botón |
| **RSA SecurID** | basada en tokens de seguridad. Genera un código que cambia constantemente, sincronizado con un servidor central RSA |

## Cómo funciona TOTP

```
   secreto compartido (al registrar)
        │
        ├──► móvil:    HMAC(secreto, tiempo/30s) → 123456
        └──► servidor: HMAC(secreto, tiempo/30s) → 123456
                                 │
                         coinciden → acceso
```

Es **offline**: el móvil no necesita red, solo el reloj sincronizado. Si el reloj del servidor
se desvía, los códigos dejan de validar (de ahí que `chrony`/`ntp` importe).

## Configurar Google Authenticator (TOTP) con SSH

1. Instalar el paquete en Linux (módulo PAM).
2. Usar la herramienta para crear un secreto (`google-authenticator`).
3. Añadir el token en la app Authenticator (en el móvil).

```bash
sudo apt install libpam-google-authenticator
google-authenticator            # genera el QR y los códigos de emergencia
```

### Paso a paso

**Configurar SSH:**

```bash
sudo vim /etc/ssh/sshd_config
```

```
UsePAM yes
KbdInteractiveAuthentication yes    # en versiones anteriores: ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

`AuthenticationMethods` es la directiva que **obliga a los dos factores**: con la coma
significa "clave pública **y después** código". Si pusieras un espacio serían alternativas.

**Configurar PAM:**

```bash
sudo vim /etc/pam.d/sshd
```

```
auth required pam_google_authenticator.so secret=/home/${USER}/.google_authenticator
```

## Flujo de una conexión con MFA

```
   ssh usuario@servidor
        │
        ├─► 1. clave pública        ✔
        │
        ├─► 2. "Verification code:" ← código del móvil
        │
        └─► shell
```

## Precauciones

- Guarda los **códigos de emergencia** (*scratch codes*) que genera el comando: son tu
  plan B si pierdes el móvil.
- Deja una sesión abierta mientras pruebas la configuración.
- Excluye de MFA las cuentas de automatización (Ansible, CI) o no podrán entrar: se hace con
  `Match` en `sshd_config`.

```
Match User ansible
    AuthenticationMethods publickey
```

> Un bastión con MFA delante y claves sin MFA por detrás es el patrón habitual: las personas
> pasan el segundo factor, las máquinas no pueden.
