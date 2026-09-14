# JAR (Java ARchive)

Archivo comprimido que contiene:

- Código Java compilado (`.class`).
- Recursos (imágenes, configs, sonidos, …).
- Metadatos (`MANIFEST.MF`).
- Dependencias (a veces, cuando es un "fat jar").

Sirve para **distribuir o ejecutar aplicaciones Java fácilmente**.

Parecido a un `.zip` o un `.tar`, pero especializado para Java.

```bash
java -jar miapp.jar
```

## Por dentro

Un JAR **es literalmente un ZIP** con una estructura concreta:

```
miapp.jar
├── META-INF/
│   └── MANIFEST.MF            ← metadatos: clase principal, classpath, versión
├── com/empresa/miapp/
│   ├── Main.class             ← bytecode (no código fuente)
│   └── servicio/Cliente.class
├── application.properties     ← recursos
└── logback.xml
```

```
   Main.java  ──javac──►  Main.class  ──jar──►  miapp.jar  ──java -jar──►  JVM la ejecuta
   (fuente)               (bytecode,              (empaquetado)              en cualquier SO
                           independiente del SO)                             con Java instalado
```

## MANIFEST.MF

```
Manifest-Version: 1.0
Main-Class: com.empresa.miapp.Main
Class-Path: lib/postgresql-42.7.3.jar lib/logback-classic-1.5.6.jar
Implementation-Version: 1.4.2
Created-By: Maven JAR Plugin 3.4.1
```

| Campo | Para qué |
|---|---|
| `Main-Class` | qué clase ejecuta `java -jar`. Sin ella: `no main manifest attribute` |
| `Class-Path` | otros JAR que necesita, con rutas **relativas al propio JAR** |
| `Implementation-Version` | versión de la aplicación (útil para saber qué hay desplegado) |

> El manifiesto es exigente: debe terminar con un salto de línea y las líneas largas se
> parten a 72 caracteres con un espacio al inicio de la continuación.

## Comandos

```bash
jar tf miapp.jar                         # listar contenido (t = table, f = file)
jar xf miapp.jar                         # extraer en la carpeta actual
jar xf miapp.jar META-INF/MANIFEST.MF    # extraer solo un fichero
jar cfe miapp.jar com.empresa.miapp.Main -C build/classes .   # crear con clase principal

# como es un ZIP, sirven las herramientas de siempre
unzip -l miapp.jar
unzip -p miapp.jar META-INF/MANIFEST.MF  # ver el manifiesto sin extraer
```

```powershell
# Windows, sin JDK
Expand-Archive .\miapp.jar -DestinationPath .\miapp-extraido   # puede requerir renombrar a .zip
```

## Ejecutar

```bash
java -jar miapp.jar                                    # usa Main-Class del manifiesto
java -cp "miapp.jar:lib/*" com.empresa.miapp.Main      # classpath explícito (Linux: ":")
java -cp "miapp.jar;lib/*" com.empresa.miapp.Main      # Windows: ";"
```

| Opción | Para qué |
|---|---|
| `-Xms512m -Xmx2g` | memoria de heap inicial / máxima |
| `-XX:MaxRAMPercentage=75` | heap como % de la memoria disponible (mejor que `-Xmx` en contenedores) |
| `-Dclave=valor` | propiedad del sistema (`-Dspring.profiles.active=prod`, `-Dfile.encoding=UTF-8`) |
| `-Djava.library.path=…` | dónde buscar librerías nativas (`.dll`/`.so`) |
| `-Duser.timezone=Europe/Madrid` | zona horaria de la JVM |

> Las opciones de la JVM van **antes** de `-jar`. Lo que va después son argumentos para la
> aplicación: `java -Xmx1g -jar app.jar --server.port=8081`.

## Tipos de empaquetado

| Tipo | Qué lleva | Cómo se ejecuta |
|---|---|---|
| **JAR normal (thin)** | solo tu código | necesita las dependencias en el classpath |
| **Fat / uber JAR** | tu código + todas las dependencias desempaquetadas dentro | `java -jar`, autocontenido |
| **Spring Boot JAR** | tu código en `BOOT-INF/classes` y dependencias como JAR dentro de `BOOT-INF/lib` | `java -jar`, con Tomcat embebido |
| **WAR** (*Web ARchive*) | app web: `WEB-INF/classes`, `WEB-INF/lib`, `web.xml` | se despliega **en un servidor** (Tomcat, ver [app Java como servicio](../windows/03-app-java-como-servicio.md)) |
| **EAR** (*Enterprise ARchive*) | varios WAR y JAR | servidores de aplicaciones Java EE (WildFly, WebLogic) |

```
   thin JAR  ── + dependencias ──► fat JAR   (java -jar, sin nada más)
   WAR       ── necesita ────────► Tomcat    (el servidor pone el HTTP)
```

## Versiones de Java: `UnsupportedClassVersionError`

Un `.class` compilado con un Java más nuevo **no arranca** en una JVM más antigua:

```
UnsupportedClassVersionError: com/empresa/miapp/Main has been compiled by a more recent
version of the Java Runtime (class file version 61.0), this version of the Java Runtime
only recognizes class file versions up to 52.0
```

| Versión de clase | Java |
|---|---|
| 52 | 8 |
| 55 | 11 |
| 61 | 17 |
| 65 | 21 |

En el ejemplo: compilado con Java 17 (61) y ejecutado con Java 8 (52). Solución: ejecutar con
Java 17 o superior, o recompilar con `--release 8`.

```bash
java -version                                   # qué JVM se está usando
unzip -p miapp.jar com/empresa/miapp/Main.class | head -c 8 | xxd   # bytes 7-8 = versión (0x3d = 61)
javap -v -cp miapp.jar com.empresa.miapp.Main | grep major
```

## Construir

| Herramienta | Comando | Resultado en |
|---|---|---|
| Maven | `mvn clean package` | `target/*.jar` |
| Gradle | `./gradlew build` | `build/libs/*.jar` |

Para publicarlos en un repositorio de artefactos (Nexus), ver
[transferencia y artefactos](../redes/03-transferencia-y-artefactos.md).

## Como servicio en Linux (systemd)

```ini
# /etc/systemd/system/miapp.service
[Unit]
Description=MiApp
After=network-online.target

[Service]
User=miapp
WorkingDirectory=/opt/miapp
ExecStart=/usr/bin/java -Xmx1g -jar /opt/miapp/miapp.jar
SuccessExitStatus=143
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

(`SuccessExitStatus=143`: la JVM sale con 143 al recibir SIGTERM en una parada normal; así
systemd no lo marca como fallo)

## En un contenedor

```dockerfile
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY target/miapp.jar app.jar
USER 1000
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/app.jar"]
```

## Seguridad: qué hay dentro de un fat JAR

Un fat JAR lleva sus dependencias **dentro**: una librería vulnerable no aparece al revisar
los paquetes del sistema. El caso famoso fue **Log4Shell** (log4j-core, 2021).

```bash
unzip -l miapp.jar | grep -i log4j-core          # Spring Boot: BOOT-INF/lib/log4j-core-2.x.jar
```

Herramientas como Trivy o Grype escanean JAR e imágenes buscando dependencias vulnerables.

```bash
jarsigner -verify -verbose miapp.jar             # comprobar la firma, si el JAR va firmado
```

## Errores típicos

| Error | Causa |
|---|---|
| `no main manifest attribute, in miapp.jar` | falta `Main-Class`: es un JAR de librería o se construyó mal |
| `ClassNotFoundException` / `NoClassDefFoundError` | falta una dependencia en el classpath (thin JAR sin `lib/`) |
| `UnsupportedClassVersionError` | Java de ejecución más antiguo que el de compilación |
| `Error: Unable to access jarfile` | ruta mal, o espacios en la ruta sin comillas |
| `OutOfMemoryError: Java heap space` | `-Xmx` insuficiente (o fuga de memoria) |
| `Address already in use` | el puerto de la app (8080) ya está ocupado |
