# Fundamentos — Base64

Base64 **codifica** la información: representa datos binarios usando solo caracteres de texto
(`A-Z`, `a-z`, `0-9`, `+`, `/`).

## El objetivo

Poder **transportar datos binarios** (una imagen, un certificado, etc.) por canales que solo
aceptan texto plano, como un JSON, un email o una URL.

## Cómo funciona

Coge los bytes de **3 en 3** (24 bits), los parte en **4 grupos de 6 bits** y cada grupo
(un valor de 0 a 63) se traduce a un carácter del alfabeto.

```
   Texto            M            a            n
   Bytes         01001101     01100001     01101110
                 └──────────────┬──────────────────┘
   Grupos de 6   010011   010110   000101   101110
   Valor           19       22        5       46
   Carácter         T        W        F        u        →  "TWFu"
```

| Valores | Caracteres |
|---|---|
| 0 – 25 | `A` – `Z` |
| 26 – 51 | `a` – `z` |
| 52 – 61 | `0` – `9` |
| 62 | `+` |
| 63 | `/` |
| relleno | `=` |

### El relleno `=`

Si los bytes no son múltiplo de 3, se completa con `=`:

| Entrada | Bytes | Base64 |
|---|---|---|
| `Man` | 3 | `TWFu` |
| `Ma` | 2 | `TWE=` |
| `M` | 1 | `TQ==` |

> Consecuencia: el resultado ocupa **un 33% más** que el original (4 caracteres por cada
> 3 bytes). Por eso incrustar ficheros grandes en base64 dentro de un JSON o un YAML no es
> buena idea.

## Variantes

| Variante | Diferencia | Dónde |
|---|---|---|
| **Estándar** | `+` y `/`, con relleno `=` | Secrets de Kubernetes, la mayoría de APIs |
| **base64url** | `-` y `_` en vez de `+` y `/`, normalmente sin `=` | JWT, URLs, nombres de fichero |
| **MIME** | salto de línea cada 76 caracteres | adjuntos de correo |
| **PEM** | salto de línea cada 64 caracteres, entre `-----BEGIN …-----` | certificados y claves |

```bash
# pasar de estándar a base64url
echo -n "datos" | base64 -w0 | tr '+/' '-_' | tr -d '='
```

## En Kubernetes

Los Secrets guardan sus valores en base64. **Solo sirve para poder meter bytes arbitrarios en
un campo de texto YAML**: no es cifrado ni ofrece ninguna seguridad.

```bash
echo -n "mi-password" | base64      # codificar
echo "bWktcGFzc3dvcmQ=" | base64 -d # decodificar
```

> Regla mental: base64 = sobre para transportar, no candado.

### `data` vs `stringData`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db
type: Opaque
data:
  password: bWktcGFzc3dvcmQ=        # ya en base64: lo codificas tú
stringData:
  usuario: app                      # en claro: Kubernetes lo codifica al guardarlo
```

```bash
# que lo codifique kubectl (sin errores de saltos de línea)
kubectl create secret generic db --from-literal=password='mi-password' \
  --dry-run=client -o yaml

# leer un valor
kubectl get secret db -o jsonpath='{.data.password}' | base64 -d

# todos los valores decodificados (con jq)
kubectl get secret db -o json | jq -r '.data | map_values(@base64d)'
```

### La trampa del salto de línea

```bash
echo -n "mi-password" | base64     # bWktcGFzc3dvcmQ=   ← correcto
echo    "mi-password" | base64     # bWktcGFzc3dvcmQK   ← incluye el "\n" final
```

Se parecen, pero la segunda contraseña es `mi-password` + salto de línea: la aplicación
falla al autenticarse y "la contraseña es la correcta". **Siempre `echo -n`** (o `printf`).

## Comandos por plataforma

| Plataforma | Codificar | Decodificar |
|---|---|---|
| Linux | `echo -n "texto" \| base64 -w0` | `echo "dGV4dG8=" \| base64 -d` |
| Fichero | `base64 -w0 cert.pem > cert.b64` | `base64 -d cert.b64 > cert.pem` |
| PowerShell | `[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("texto"))` | `[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("dGV4dG8="))` |
| Windows CMD | `certutil -encode entrada.bin salida.b64` | `certutil -decode salida.b64 entrada.bin` |
| Python | `base64.b64encode(b"texto").decode()` | `base64.b64decode("dGV4dG8=")` |
| Ansible (Jinja2) | `{{ texto \| b64encode }}` | `{{ texto \| b64decode }}` |

- `-w0` (GNU) evita que `base64` parta la salida cada 76 caracteres, que rompería un YAML.
- `certutil -encode` añade las líneas `-----BEGIN CERTIFICATE-----` / `-----END…`.
- El módulo `ansible.builtin.slurp` devuelve el contenido del fichero remoto **en base64**:
  `{{ resultado.content | b64decode }}`.

### PowerShell `-EncodedCommand`

PowerShell acepta un comando entero en base64, pero de su texto en **UTF-16LE**, no UTF-8:

```powershell
$cmd = 'Write-Output hola'
$b64 = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd))   # Unicode = UTF-16LE
powershell.exe -NoProfile -EncodedCommand $b64
```

Útil para pasar scripts con comillas complicadas por WinRM o una tarea programada. (También
es una técnica muy usada por malware, así que los antivirus y EDR lo vigilan.)

## Reconocerlo a simple vista

| Empieza por | Suele ser |
|---|---|
| `eyJ` | JSON que empieza por `{"`: típico de un **JWT** (`eyJhbGciOi…`) |
| `LS0tLS1` | `-----`: un **PEM** (certificado o clave) codificado otra vez en base64, como en un kubeconfig |
| `H4sI` | datos comprimidos con **gzip** |
| `TVqQ` | un **ejecutable de Windows** (cabecera `MZ`) |
| Acaba en `=` o `==` | relleno de base64 |

```bash
# un JWT son tres partes base64url separadas por puntos: cabecera.payload.firma
echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null; echo
```

> Leer el payload de un JWT **no requiere ninguna clave**: cualquiera que tenga el token ve su
> contenido. La firma solo impide **modificarlo**. No metas datos sensibles en un JWT.

## Dónde aparece en el día a día

| Sitio | Qué va en base64 |
|---|---|
| Secrets de Kubernetes | los valores de `data:` |
| kubeconfig | `certificate-authority-data`, `client-certificate-data`, `client-key-data` |
| Cabecera HTTP Basic | `Authorization: Basic dXNlcjpwYXNz` (`user:pass`) |
| JWT / OIDC | cabecera y payload (base64url) |
| Certificados PEM | el cuerpo entre `BEGIN` y `END` |
| cloud-init / user-data | scripts de arranque de VMs en la nube |
| Correo | adjuntos (MIME) |
| Data URIs | `data:image/png;base64,iVBORw0…` |

```bash
# ver el certificado de la CA de un kubeconfig
grep certificate-authority-data ~/.kube/config | awk '{print $2}' | base64 -d \
  | openssl x509 -noout -subject -enddate
```

## Base64 no es seguridad

Quien pueda leer un Secret (`kubectl get secret -o yaml`) tiene la contraseña: decodificar
es un comando. Para proteger secretos de verdad:

| Opción | Qué hace |
|---|---|
| RBAC restrictivo sobre `secrets` | limitar quién puede leerlos (ver [RBAC](../kubernetes/09-rbac.md)) |
| *Encryption at rest* de etcd | cifra los Secrets en el disco del control plane |
| **Sealed Secrets** | Secrets cifrados que sí se pueden guardar en Git |
| **SOPS** | cifra valores dentro de YAML/JSON con age, PGP o un KMS |
| **External Secrets Operator** + Vault/OpenBao | los secretos viven en Vault y se sincronizan al clúster (ver [Vault](../vault/01-vault-kv.md)) |
