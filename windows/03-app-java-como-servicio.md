# Windows — Instalar una app Java como servicio (Tomcat)

Procedimiento genérico para dejar una aplicación web Java corriendo como servicio de Windows.

## Qué hay por debajo

```
   Servicios de Windows (services.msc)
        │
        ▼
   MiAppWeb  ──►  tomcat9.exe   (Apache Commons Daemon "procrun":
                     │           adapta una app Java al modelo de servicios)
                     ▼
                  jvm.dll        (la JVM de Java 1.8.0\, cargada dentro del proceso)
                     │
                     ▼
                  Tomcat  ──►  webapps\MiAppWeb.war  ──►  http://servidor:8080/MiAppWeb
```

| Fichero en `Tomcat 9\bin` | Para qué |
|---|---|
| `service.bat` | instala o desinstala el servicio |
| `tomcat9.exe` | el ejecutable del servicio (procrun) |
| `tomcat9w.exe` | ventana de configuración del servicio (memoria, JVM, cuenta, arranque) |
| `startup.bat` / `shutdown.bat` | arrancar Tomcat **en consola**, sin servicio (para probar) |

## 1. Estructura de carpetas

Crear en la raíz del disco:

```
C:\apps\
└── MiAppWeb\
    ├── Java 1.8.0\      # JRE/JDK que usará la app
    └── Tomcat 9\        # contenedor de servlets
```

Dentro de Tomcat, lo que importa:

```
Tomcat 9\
├── bin\        ejecutables y service.bat
├── conf\       server.xml (puertos), context.xml, tomcat-users.xml
├── lib\        librerías compartidas (drivers JDBC)
├── logs\       catalina, access logs y logs del servicio
├── webapps\    aquí va el .war (ROOT.war = se sirve en "/")
├── temp\
└── work\       JSP compiladas (se puede borrar con el servicio parado)
```

> Que Java viva **dentro** de la carpeta de la app fija la versión exacta que usa: una
> actualización de Java del sistema no la rompe.

## 2. Instalar el servicio

Desde consola (`CMD`), ir primero a la ruta de binarios de Tomcat:

```cmd
cd C:\apps\MiAppWeb\Tomcat 9\bin
```

Y ya dentro de la ruta:

```cmd
service install MiAppWeb
```

Antes, en la misma consola (como **administrador**), indicar qué Java usar:

```cmd
set "JAVA_HOME=C:\apps\MiAppWeb\Java 1.8.0"
set "CATALINA_HOME=C:\apps\MiAppWeb\Tomcat 9"
cd /d "C:\apps\MiAppWeb\Tomcat 9\bin"
service.bat install MiAppWeb
```

> **Rutas con espacios**: en CMD `cd` las tolera, pero en `set` y en casi todo lo demás hay
> que usar comillas. Si puedes elegir, evita los espacios (`Java8`, `Tomcat9`).
>
> **Desde PowerShell** hay que escribir `.\service.bat install MiAppWeb`: PowerShell no
> ejecuta scripts de la carpeta actual sin `.\`.

## 3. Ajustar el servicio

`service.bat` suele dejar el servicio con arranque **manual** y la memoria por defecto.

### Con la ventana gráfica

```cmd
tomcat9w.exe //ES//MiAppWeb
```

| Pestaña | Qué se configura |
|---|---|
| General | tipo de inicio (**Automatic**) |
| Log On | cuenta con la que corre |
| Java | ruta de `jvm.dll`, `-Xms`/`-Xmx`, opciones `-D` |
| Startup / Shutdown | clases de arranque y timeouts |

### Por línea de comandos (repetible, para scripts)

```cmd
tomcat9.exe //US//MiAppWeb --Startup auto --JvmMs 512 --JvmMx 2048 ++JvmOptions "-Dfile.encoding=UTF-8"
```

| Opción | Significado |
|---|---|
| `//US//nombre` | *Update Service* |
| `--Startup auto` | inicio automático |
| `--JvmMs` / `--JvmMx` | memoria inicial / máxima en **MB** |
| `--JvmOptions` | **reemplaza** las opciones de la JVM |
| `++JvmOptions` | **añade** opciones a las existentes |

### Arrancar y comprobar

```powershell
Set-Service MiAppWeb -StartupType Automatic
Start-Service MiAppWeb
Get-Service MiAppWeb

Get-CimInstance Win32_Service -Filter "Name='MiAppWeb'" |
    Select-Object Name, State, StartMode, StartName, PathName

Test-NetConnection localhost -Port 8080
Invoke-WebRequest http://localhost:8080/MiAppWeb/ -UseBasicParsing | Select-Object StatusCode
```

### Reinicio automático si cae

```cmd
sc.exe failure MiAppWeb reset= 86400 actions= restart/60000/restart/60000/restart/60000
sc.exe qfailure MiAppWeb
```

(el espacio después de `reset=` y `actions=` es obligatorio en `sc.exe`)

## 4. Desplegar la aplicación

```powershell
Stop-Service MiAppWeb
Remove-Item 'C:\apps\MiAppWeb\Tomcat 9\webapps\MiAppWeb' -Recurse -Force   # carpeta expandida antigua
Copy-Item .\MiAppWeb.war 'C:\apps\MiAppWeb\Tomcat 9\webapps\'
Start-Service MiAppWeb
```

| Nombre del `.war` | URL |
|---|---|
| `MiAppWeb.war` | `http://servidor:8080/MiAppWeb` |
| `ROOT.war` | `http://servidor:8080/` |

## Logs

| Fichero en `logs\` | Qué contiene |
|---|---|
| `commons-daemon.AAAA-MM-DD.log` | el arranque del servicio en sí (errores de JVM, rutas) |
| `miappweb-stderr.AAAA-MM-DD.log` / `-stdout` | salida de consola de Tomcat y la app |
| `catalina.AAAA-MM-DD.log` | log principal de Tomcat |
| `localhost.AAAA-MM-DD.log` | errores de despliegue de las aplicaciones |
| `localhost_access_log.AAAA-MM-DD.txt` | peticiones HTTP |

```powershell
Get-Content 'C:\apps\MiAppWeb\Tomcat 9\logs\catalina.2026-09-14.log' -Wait -Tail 50
```

## Desinstalar

```cmd
sc.exe stop MiAppWeb
service.bat remove MiAppWeb
```

## Notas

- El nombre que se pasa a `service install` es el nombre con el que aparecerá el servicio.
- Después se puede gestionar con `Get-Service` / `Start-Service`, o desde Ansible con
  `ansible.windows.win_service`.
- Conviene comprobar que `JAVA_HOME` apunta al JRE de la carpeta correcta antes de instalar.
- Revisa con qué **cuenta** corre (suele quedar como LocalSystem). En producción, mejor una
  cuenta de servicio dedicada o una gMSA, con permisos solo sobre `C:\apps\MiAppWeb`.

### Con Ansible

```yaml
- name: Asegurar el servicio MiAppWeb arrancado y automático
  ansible.windows.win_service:
    name: MiAppWeb
    start_mode: auto
    state: started

- name: Esperar a que responda
  ansible.windows.win_wait_for:
    port: 8080
    timeout: 120
```

## Alternativas para apps que no son Tomcat

Una app **Spring Boot** empaquetada como fat jar (`java -jar app.jar`, ver
[JAR](../herramientas/jar.md)) no necesita Tomcat externo, pero sí un "envoltorio" de servicio:

| Herramienta | Cómo |
|---|---|
| **WinSW** | un `.exe` + un `.xml` con el comando `java -jar`; se instala con `winsw install` |
| **NSSM** | `nssm install MiApp "C:\Java\bin\java.exe" "-jar C:\apps\app.jar"` |
| **procrun** | el mismo que usa Tomcat, configurable para cualquier clase Java |

> No sirve `sc.exe create` apuntando a `java.exe` directamente: `java.exe` no habla el
> protocolo de servicios de Windows y el servicio falla a los 30 segundos con el error 1053.

## Errores típicos

| Síntoma | Causa habitual |
|---|---|
| El servicio arranca y se para al instante | `JAVA_HOME` mal en la instalación: mira `commons-daemon.*.log` |
| `Failed creating java …\jvm.dll` | la ruta a `jvm.dll` no existe, o **mezcla 32/64 bits** (Tomcat de 64 bits con Java de 32, o al revés) |
| `Error 1053: el servicio no respondió a tiempo` | arranque lento de la app o JVM que no carga; revisar logs |
| `Address already in use: bind` / puerto 8080 ocupado | otro proceso usa el puerto: `netstat -ano \| findstr :8080` y `tasklist /fi "PID eq <pid>"` |
| La app no aparece | error de despliegue: `localhost.*.log` |
| `Access denied` escribiendo en `logs\` o `temp\` | la cuenta del servicio no tiene permisos sobre la carpeta |
| `OutOfMemoryError: Java heap space` | subir `--JvmMx` (y revisar si es una fuga de memoria) |
| Caracteres raros (`Ã±`) en la app o los logs | añadir `-Dfile.encoding=UTF-8` (ver [codificación](../fundamentos/parsers-y-codificacion.md)) |
