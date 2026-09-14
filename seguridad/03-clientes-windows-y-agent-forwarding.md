# SSH — Clientes en Windows y agent forwarding

## WinSCP (Windows Secure Copy)

Interfaz gráfica para transferir archivos.

```
   ┌──────────────────────┬──────────────────────┐
   │  TU ORDENADOR        │   SERVIDOR REMOTO    │
   │  C:\proyectos        │   /opt/app           │
   │                      │                      │
   │  build.zip  ────────────► arrastrar y soltar│
   └──────────────────────┴──────────────────────┘
```

- **Protocolos que soporta**: SFTP, SCP, FTP.
- **Ventajas**:
  - Gestión de sitios (favoritos): puedes guardar IPs, usuarios, etc.
  - Editor de texto integrado: doble clic sobre el archivo y lo edita en el servidor.
  - Seguridad con claves SSH: soporta claves privadas, incluidos archivos `.ppk` de PuTTY.
  - Sincronización de carpetas y comparación de directorios.

## PuTTY

Cliente SSH más famoso para Windows (la ventana negra). Es el programa principal de la suite.

### Herramientas de la suite

| Herramienta | Para qué |
|---|---|
| `putty.exe` | cliente SSH/Telnet interactivo |
| `puttygen.exe` | generar y convertir claves |
| `pageant.exe` | agente de autenticación (guarda la clave desbloqueada) |
| `plink.exe` | PuTTY en línea de comandos, para scripts |
| `pscp.exe` / `psftp.exe` | equivalentes de scp y sftp |

**PuTTYgen** (key generator)

| Clave | Detalle |
|---|---|
| **pública** | el candado. Cualquiera puede verla |
| **privada** | la llave (archivo `.ppk`). Nunca se comparte |

> El formato `.ppk` es propio de PuTTY. Para usar una clave de OpenSSH en PuTTY (o al revés)
> hay que convertirla: *PuTTYgen → Load → Conversions → Export OpenSSH key*.

**Pageant** (authentication agent)

- Evita tener que poner la contraseña de la clave una y otra vez.
- Abres Pageant y cargas tu clave privada; queda en memoria mientras la sesión esté abierta.

## Cómo funciona un agente SSH

```
   ┌──────────────────────────────┐
   │  ssh-agent / Pageant         │  guarda la clave PRIVADA desbloqueada
   │  (en memoria, nunca en disco)│
   └──────────────┬───────────────┘
                  │ firma los retos
                  ▼
   ssh ──────────────────────────────► servidor
        (no vuelve a pedir passphrase)
```

## Agent forwarding

Permite usar tu clave local desde un servidor intermedio **sin copiar la clave privada allí**.

```
   TU PC                 BASTIÓN                 SERVIDOR FINAL
   ┌───────┐  ssh -A     ┌────────┐  ssh         ┌────────────┐
   │ agente│────────────►│        │─────────────►│            │
   │ clave │◄── reto ────┤ reenvía│◄── reto ─────┤ verifica   │
   └───────┘   firmado   └────────┘   firmado    └────────────┘
   la clave privada NUNCA sale de tu PC
```

### Pasos

1. En el servidor: `AllowAgentForwarding yes` (en `sshd_config`).
2. En el cliente: activarlo (`ForwardAgent yes` o `ssh -A`).
3. Tener el agente corriendo con la clave cargada (Pageant en Windows).

### En Linux

```bash
vim ~/.ssh/config
```

```
Host bastion
    HostName bastion.interno.local
    ForwardAgent yes
```

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l                 # ver claves cargadas
ssh-add -D                 # descargar todas
```

### Riesgo y alternativa

> Quien tenga **root en el servidor intermedio** puede usar tu agente mientras estés conectado
> (no puede robar la clave, pero sí firmar con ella). Por eso hoy se prefiere **ProxyJump**,
> que no expone el agente:

```
Host final
    HostName 10.0.0.20
    ProxyJump bastion
```

```bash
ssh -J bastion usuario@10.0.0.20     # equivalente en línea de comandos
```

```
   ForwardAgent → el bastión puede pedirle firmas a tu agente
   ProxyJump    → el bastión solo reenvía la conexión cifrada (túnel), no ve nada
```
