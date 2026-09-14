# Windows — DLLs

**DLL** = *Dynamic Link Library*.

Contiene **código reutilizable** que pueden usar varios programas al mismo tiempo.
En vez de que cada programa incluya todo su código dentro del `.exe`, Windows permite
separar funcionalidades en DLLs.

## Qué pueden contener

Funciones, clases, drivers, recursos, iconos, APIs…

## Ventajas

| Ventaja | Detalle |
|---|---|
| **Reutilización** | muchos programas usan las mismas DLLs |
| **Menor tamaño** | el `.exe` pesa menos |
| **Actualizaciones** | puedes actualizar la DLL sin recompilar toda la aplicación |
| **Modularidad** | separa funcionalidades |

## Enlace estático vs dinámico

```
   ESTÁTICO                                DINÁMICO (DLL)
   ────────                                ──────────────
   ┌──────────── app.exe ────────┐         ┌──── app.exe ────┐     ┌─────────────┐
   │ código propio               │         │ código propio   │────►│ libcrypto   │
   │ + copia de la librería      │         └─────────────────┘     │ .dll        │
   └─────────────────────────────┘         ┌──── otra.exe ───┐────►│ (una copia  │
   autocontenido, más grande,              │ código propio   │     │ en disco y  │
   hay que recompilar para                 └─────────────────┘     │ en memoria) │
   parchear la librería                                            └─────────────┘
```

| En otros sistemas | Extensión | Ver dependencias |
|---|---|---|
| Windows | `.dll` | `dumpbin /dependents app.exe`, Dependencies |
| Linux | `.so` (*shared object*) | `ldd /usr/bin/curl` |
| macOS | `.dylib` | `otool -L app` |

## Cómo busca Windows una DLL

Cuando un programa pide `libreria.dll` sin ruta, Windows la busca en este orden (con el
modo de búsqueda segura activado, que es el valor por defecto):

```
   0. ¿ya está cargada en memoria? ¿es una "KnownDLL" del sistema?
   1. la carpeta del .exe
   2. C:\Windows\System32
   3. C:\Windows\System            (16 bits, legado)
   4. C:\Windows
   5. el directorio actual
   6. las carpetas del PATH, en orden
```

Consecuencias prácticas:

- Una DLL **junto al `.exe`** gana a la del sistema: así una app lleva "su" versión.
- Si dos apps ponen versiones distintas de la misma DLL en el `PATH`, gana la que aparezca
  antes: origen de errores que "solo pasan en este servidor".
- Las *KnownDLLs* (`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs`) siempre
  se cargan desde System32.

## System32 y SysWOW64: la trampa del nombre

En Windows de **64 bits**:

| Carpeta | Contiene |
|---|---|
| `C:\Windows\System32` | DLLs de **64 bits** (sí, a pesar del "32") |
| `C:\Windows\SysWOW64` | DLLs de **32 bits** (WOW64 = *Windows 32-bit on Windows 64-bit*) |

Un proceso de 32 bits que pide `System32` es **redirigido** automáticamente a `SysWOW64`.

> Un proceso **no puede cargar una DLL de otra arquitectura**: un `.exe` de 64 bits no carga
> una DLL de 32 bits. Es el origen de muchos errores con drivers ODBC, JNI y plugins.

## DLL Hell

```
   App A instala  msvcr.dll v1  ─┐
                                 ├──► misma ruta, gana la última
   App B instala  msvcr.dll v2  ─┘         │
                                           ▼
                           App A deja de funcionar
```

Soluciones que ha ido introduciendo Windows:

- **Side-by-side (WinSxS)**: varias versiones conviven en `C:\Windows\WinSxS` y cada app declara
  en su *manifest* cuál necesita.
- **Redistribuibles de Visual C++**: instaladores oficiales de las librerías de ejecución.
- **Aplicaciones autocontenidas**: llevan sus DLLs en su propia carpeta.

## Errores típicos

| Error | Causa / solución |
|---|---|
| `The code execution cannot proceed because MSVCP140.dll / VCRUNTIME140.dll was not found` | falta el **Visual C++ Redistributable**: instalar el oficial de Microsoft (x64 **y** x86 si hay apps de 32 bits) |
| `0xc000007b` al arrancar | mezcla de arquitecturas: una DLL de 32 bits en un proceso de 64 o al revés |
| `%1 is not a valid Win32 application` | lo mismo, o un fichero corrupto |
| `The specified module could not be found` | la DLL existe pero **una de sus dependencias** no: mirar con Dependencies o Process Monitor |
| Java: `UnsatisfiedLinkError: Can't load IA 32-bit .dll on a AMD 64-bit platform` | librería nativa de 32 bits con una JVM de 64 |
| Java: `no xxx in java.library.path` | la DLL no está en `-Djava.library.path` ni en el `PATH` |
| .NET: `BadImageFormatException` | ensamblado compilado para otra arquitectura (x86 vs x64) |
| Error en DLLs del propio Windows | ficheros de sistema dañados: `sfc /scannow` y, si falla, `DISM /Online /Cleanup-Image /RestoreHealth` |

> **Nunca** descargues DLLs sueltas de webs tipo "dll-download": son una vía clásica de
> malware. Instala el paquete oficial que la contiene.

## Herramientas

| Herramienta | Para qué |
|---|---|
| **Dependencies** (proyecto open source, sucesor de *Dependency Walker*) | ver el árbol de DLLs que necesita un `.exe` y cuáles faltan |
| `dumpbin /dependents app.exe` | lo mismo en consola (viene con Visual Studio / Build Tools) |
| **Process Explorer** (Sysinternals) | qué DLLs tiene cargadas un proceso en ejecución |
| **Process Monitor** (Sysinternals) | ver en directo dónde **busca** una DLL (filtrar por `NAME NOT FOUND`) |
| `tasklist /m` | módulos cargados por proceso |

```cmd
tasklist /m jvm.dll                        :: qué procesos tienen cargada esta DLL
tasklist /m /fi "IMAGENAME eq tomcat9.exe" :: qué DLLs usa un proceso
where libcrypto-3-x64.dll                  :: ¿hay alguna en el PATH? (where.exe desde PowerShell)
```

```powershell
(Get-Process tomcat9).Modules | Select-Object ModuleName, FileName, FileVersion
(Get-Item C:\Windows\System32\kernel32.dll).VersionInfo | Select-Object FileVersion, ProductVersion
```

> Una DLL cargada por un proceso **está bloqueada**: no se puede sobrescribir ni borrar hasta
> parar ese proceso. Por eso los parches de Windows y muchas actualizaciones piden reiniciar.

## DLLs COM y `regsvr32`

Algunas DLLs son componentes **COM** y hay que registrarlas en el registro para que otros
programas las encuentren por su identificador (CLSID), no por su ruta:

```cmd
regsvr32 C:\apps\componente.dll          :: registrar
regsvr32 /u C:\apps\componente.dll       :: desregistrar
C:\Windows\SysWOW64\regsvr32 comp32.dll  :: registrar una DLL COM de 32 bits
```

> Solo aplica a DLLs COM. `regsvr32` sobre una DLL normal da el error "the entry-point
> DllRegisterServer was not found", y es lo esperado.

## Seguridad: DLL hijacking

Como Windows busca primero en la carpeta del `.exe`, si un atacante puede **escribir** en esa
carpeta, puede dejar una DLL con el nombre esperado y el programa la cargará con sus
privilegios.

- Las aplicaciones deben instalarse en carpetas donde los usuarios normales **no tengan
  escritura** (`C:\Program Files` lo garantiza; `C:\apps\…` hay que configurarlo).
- Si un servicio corre como LocalSystem y su carpeta es escribible por usuarios, es una
  escalada de privilegios servida.

```powershell
icacls C:\apps\MiAppWeb        # revisar que "Users" / "Usuarios" no tenga (M) ni (W)
```
