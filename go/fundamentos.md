# Go (Golang)

Lenguaje de programación de código abierto desarrollado por Google en 2007 y lanzado en 2009.
Creado para resolver problemas de Google: sistemas masivos, redes complejas y bases de código
gigantescas.

## Características

| Característica | Detalle |
|---|---|
| **Simplicidad extrema** | sintaxis limpia, minimalista y directa |
| **Velocidad de ejecución** | lenguaje compilado (se traduce directamente a código máquina), muy rápido, como C o C++ |
| **Concurrencia nativa** | su "superpoder". Diseñado para procesadores con múltiples núcleos. Utiliza **goroutines**: funciones que pueden ejecutarse simultáneamente consumiendo poca memoria |
| **Tipado estático** | los errores se detectan antes de ejecutar (en tiempo de compilación), evita sorpresas en producción |
| **Recolector de basura** (garbage collector) | gestión automática de la memoria: no hay que liberar memoria manualmente como en C/C++ |
| **Un único binario** | compila a un ejecutable sin dependencias: se copia al servidor y funciona |
| **Compilación cruzada** | desde Windows se genera el binario de Linux ARM con dos variables de entorno |

## Usos principales

- ⭐ **Rey de la infraestructura moderna y el desarrollo backend.**
- **Servicios en la nube y DevOps**: Docker, Kubernetes y Terraform están escritos en Go.
- **Desarrollo backend y APIs**: creación de servicios web de alto rendimiento que necesitan
  procesar millones de solicitudes por segundo.
- **Redes y sistemas**: eficiente y con gran manejo de protocolos, ideal para herramientas de
  red.
- **Microservicios**: su velocidad de inicio y su bajo consumo de recursos lo hacen perfecto
  para arquitecturas fragmentadas.

### Herramientas del día a día escritas en Go

| Área | Herramientas |
|---|---|
| Contenedores | Docker, containerd, runc, Podman |
| Kubernetes | Kubernetes, etcd, Helm, k9s, kind, MetalLB, cert-manager, Argo CD, Flux |
| Infraestructura como código | Terraform, OpenTofu, Packer |
| Secretos y red | Vault, OpenBao, Consul, CoreDNS, Traefik, Caddy |
| Observabilidad | Prometheus, node_exporter, Alertmanager, Loki, backend de Grafana |

> Contraste: **Ansible** y **Molecule** están escritos en Python. Por eso Ansible necesita un
> intérprete de Python en el nodo de control, mientras que `kubectl` o `terraform` son un
> único binario que se descarga y funciona.

## Otros detalles

- **"Gopher"**: la mascota oficial. A los desarrolladores de Go se les suele llamar así.
- **Manejo de errores**: las funciones devuelven el resultado y el error al mismo tiempo, lo
  que obliga a escribir constantemente:

```go
if err != nil {
    return err
}
```

  Repetitivo, pero sabes exactamente dónde se produjo el error.

- **Formateo estricto por obligación**: incluye la herramienta `gofmt`, que formatea el código
  automáticamente bajo un único estándar.
- **Genéricos**: antes Go no tenía genéricos, lo que obligaba a repetir mucho código; se
  añadieron en la versión 1.18.
- **Librería estándar brutal**: a diferencia de JavaScript (Node.js), donde necesitas instalar
  cientos de paquetes externos, Go viene con una librería estándar muy potente. Puedes crear un
  servidor web seguro, manejar JSON y gestionar criptografía avanzada sin instalar nada externo.
- Los desarrolladores de Go están bien cotizados: mucha demanda en empresas con grandes
  volúmenes de datos.

## Herramienta `go`

| Comando | Qué hace |
|---|---|
| `go mod init gitlab.lab.local/devops/healthcheck` | crea un módulo (`go.mod`) |
| `go run .` | compila y ejecuta sin dejar binario |
| `go build -o bin/healthcheck .` | compila a un ejecutable |
| `go test ./...` | ejecuta todos los tests |
| `go vet ./...` | análisis estático: errores sospechosos que compilan |
| `go fmt ./...` | formatea todo el código |
| `go get github.com/org/lib@v1.2.3` | añade o actualiza una dependencia |
| `go mod tidy` | limpia `go.mod`/`go.sum`: añade lo que falta y quita lo que sobra |
| `go install github.com/org/herramienta@latest` | instala un binario en `$GOPATH/bin` |
| `go env` | configuración (`GOOS`, `GOARCH`, `GOPROXY`…) |

```
healthcheck/
├── go.mod          # nombre del módulo, versión de Go y dependencias
├── go.sum          # hashes de las dependencias (se commitea)
├── main.go
├── checks.go
└── checks_test.go  # los tests viven junto al código, en ficheros *_test.go
```

## Sintaxis esencial

```go
package main

import (
	"errors"
	"fmt"
	"os"
)

// struct: agrupa datos. Las etiquetas `json:` controlan la serialización
type Servidor struct {
	Nombre string `json:"nombre"`
	IP     string `json:"ip"`
	Puerto int    `json:"puerto"`
}

// método: función asociada a un tipo
func (s Servidor) Direccion() string {
	return fmt.Sprintf("%s:%d", s.IP, s.Puerto)
}

// interfaz: cualquier tipo que tenga Comprobar() error la cumple, sin declararlo
type Comprobador interface {
	Comprobar() error
}

func leerConfig(ruta string) ([]byte, error) {
	datos, err := os.ReadFile(ruta)
	if err != nil {
		return nil, fmt.Errorf("leyendo %s: %w", ruta, err) // %w envuelve el error original
	}
	return datos, nil
}

func main() {
	nombre := "web01"          // := declara e infiere el tipo
	var puertos []int          // slice (lista dinámica) vacío
	puertos = append(puertos, 80, 443)

	entornos := map[string]string{"web01": "prod", "web-dev01": "dev"}

	for i, p := range puertos { // for es el ÚNICO bucle (no hay while)
		fmt.Println(i, p)
	}

	if env, ok := entornos[nombre]; ok {
		fmt.Println(nombre, "está en", env)
	}

	s := Servidor{Nombre: nombre, IP: "10.0.0.15", Puerto: 443}
	fmt.Println(s.Direccion())

	_, err := leerConfig("/etc/noexiste.yml")
	if errors.Is(err, os.ErrNotExist) { // se puede comprobar el error original aunque esté envuelto
		fmt.Println("no hay fichero de configuración:", err)
	}
}
```

| Concepto | Nota |
|---|---|
| Mayúscula inicial | `Servidor`, `Direccion` son **públicos** (exportados); en minúscula, privados al paquete |
| `defer` | ejecuta algo al salir de la función: `defer f.Close()` |
| `nil` | valor vacío de punteros, slices, maps, interfaces y errores |
| Sin excepciones | los errores son valores que se devuelven; `panic` solo para lo irrecuperable |
| Variables sin usar | **no compila**: obliga a mantener el código limpio |

## Concurrencia: goroutines y canales

```
   main ──┬── go comprobar(nexus)   ──┐
          ├── go comprobar(grafana) ──┼──► canal "resultados" ──► main imprime
          └── go comprobar(vault)   ──┘
   las tres peticiones van en PARALELO: tarda lo que la más lenta, no la suma
```

```go
package main

import (
	"fmt"
	"net/http"
	"sync"
	"time"
)

func main() {
	urls := []string{
		"https://nexus.lab.local",
		"https://grafana.lab.local/api/health",
		"https://vault.lab.local:8200/v1/sys/health",
	}

	resultados := make(chan string)
	var wg sync.WaitGroup
	cliente := &http.Client{Timeout: 5 * time.Second}

	for _, u := range urls {
		wg.Add(1)
		go func(url string) { // "go" lanza la función en una goroutine
			defer wg.Done()
			resp, err := cliente.Get(url)
			if err != nil {
				resultados <- fmt.Sprintf("✘ %s: %v", url, err)
				return
			}
			defer resp.Body.Close()
			resultados <- fmt.Sprintf("✔ %s: %d", url, resp.StatusCode)
		}(u)
	}

	go func() {
		wg.Wait()         // cuando terminen todas…
		close(resultados) // …se cierra el canal y el for de abajo acaba
	}()

	for r := range resultados {
		fmt.Println(r)
	}
}
```

| Pieza | Para qué |
|---|---|
| `go f()` | lanzar en paralelo (una goroutine ocupa unos pocos KB) |
| `chan T` | canal: tubería tipada para pasar datos entre goroutines |
| `sync.WaitGroup` | esperar a que terminen N goroutines |
| `sync.Mutex` | proteger datos compartidos |
| `context.Context` | cancelar y poner timeouts a operaciones en cadena |

> Lema de Go: *"No comuniques compartiendo memoria; comparte memoria comunicando"*.
> `go run -race .` detecta accesos concurrentes peligrosos.

## Un servicio HTTP mínimo (solo librería estándar)

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"os"
	"time"
)

var version = "dev" // se sobrescribe al compilar con -ldflags

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /health", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"status": "ok", "version": version})
	})

	addr := ":" + getenv("PORT", "8080")
	srv := &http.Server{Addr: addr, Handler: mux, ReadHeaderTimeout: 5 * time.Second}

	log.Printf("escuchando en %s", addr)
	log.Fatal(srv.ListenAndServe())
}

func getenv(clave, porDefecto string) string {
	if v := os.Getenv(clave); v != "" {
		return v
	}
	return porDefecto
}
```

(`"GET /health"` con el método en la ruta requiere Go 1.22 o superior)

## Compilar para otros sistemas

```bash
GOOS=linux   GOARCH=amd64 go build -o dist/healthcheck-linux-amd64 .
GOOS=linux   GOARCH=arm64 go build -o dist/healthcheck-linux-arm64 .     # Raspberry Pi, Graviton
GOOS=windows GOARCH=amd64 go build -o dist/healthcheck.exe .
```

```powershell
# lo mismo desde PowerShell
$env:GOOS = "linux"; $env:GOARCH = "amd64"; go build -o dist/healthcheck .
```

```bash
# binario estático, sin símbolos de depuración y con la versión incrustada
CGO_ENABLED=0 go build -ldflags="-s -w -X main.version=1.4.2" -o healthcheck .
```

| Flag | Efecto |
|---|---|
| `CGO_ENABLED=0` | sin dependencias de C: binario 100% estático (funciona en `scratch` o distroless) |
| `-ldflags="-s -w"` | quita tablas de depuración: binario más pequeño |
| `-X main.version=…` | asigna una variable de tipo string en tiempo de compilación |

## Imagen Docker mínima

```dockerfile
FROM golang:1.24 AS build
WORKDIR /src
COPY go.mod go.sum* ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /healthcheck .

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /healthcheck /healthcheck
EXPOSE 8080
ENTRYPOINT ["/healthcheck"]
```

```
   imagen de build (golang)  ~800 MB   →  se descarta
   imagen final (distroless) ~2 MB + tu binario, sin shell, usuario no root
```

(ver [Dockerfile y multi-stage](../docker/01-fundamentos-y-dockerfile.md))

## Tests

```go
// checks_test.go
package main

import "testing"

func TestDireccion(t *testing.T) {
	s := Servidor{IP: "10.0.0.15", Puerto: 443}
	if got, want := s.Direccion(), "10.0.0.15:443"; got != want {
		t.Errorf("Direccion() = %q, se esperaba %q", got, want)
	}
}
```

```bash
go test ./...            # todos
go test -run Direccion -v
go test -cover ./...     # cobertura
```

## Genéricos

```go
func Filtrar[T any](lista []T, cumple func(T) bool) []T {
	var resultado []T
	for _, v := range lista {
		if cumple(v) {
			resultado = append(resultado, v)
		}
	}
	return resultado
}

// uso, dentro de una función:
https := Filtrar(servidores, func(s Servidor) bool { return s.Puerto == 443 })
```

## Dependencias privadas

```bash
# módulos del GitLab interno: no pasar por el proxy público ni por la base de datos de checksums
go env -w GOPRIVATE=gitlab.lab.local/*

# proxy de módulos propio (Nexus tiene repositorios de tipo Go)
go env -w GOPROXY=https://nexus.lab.local/repository/go-proxy/,direct
```

## Go vs Python para herramientas de operaciones

| | Go | Python |
|---|---|---|
| Distribución | un binario, sin nada instalado | intérprete + dependencias (venv, pip) |
| Velocidad | alta | suficiente para scripts; lenta en CPU intensiva |
| Concurrencia | nativa y sencilla | asyncio / threads / multiprocessing |
| Rapidez para escribir un script | media (más ceremonia) | alta |
| Ecosistema de datos | limitado | enorme (pandas, etc.) |
| Encaja en… | CLIs, exporters, operadores de Kubernetes, servicios | automatización rápida, Ansible, análisis de datos |
