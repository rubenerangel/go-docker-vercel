# Go + Docker multi-stage → Vercel desde la terminal

![Go](https://img.shields.io/badge/Go-1.24-00ADD8?logo=go&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-multi--stage-2496ED?logo=docker&logoColor=white)
![Alpine](https://img.shields.io/badge/Alpine-3.20-0D597F?logo=alpinelinux&logoColor=white)
![Vercel](https://img.shields.io/badge/Deploy-Vercel%20CLI-000000?logo=vercel&logoColor=white)

Proyecto de aprendizaje: un servidor HTTP escrito en **Go**, empaquetado con un **Dockerfile multi-stage** (`builder` + `runner`) y desplegado **directamente a Vercel desde la línea de comandos**, sin pasar por GitHub, sin CI y sin panel web.

**Demo en vivo:** https://05-vercel-bice.vercel.app

El objetivo no era la app (es un "hello world" a propósito), sino dominar el camino completo: **código → imagen optimizada → contenedor corriendo en producción** con un solo comando.

## Qué demuestra este proyecto

| Área | Qué se aplica |
| --- | --- |
| Go | Servidor `net/http` sin dependencias externas, puerto configurable vía `PORT` (12-factor) |
| Docker | Build multi-stage con etapas nombradas, separación entre toolchain de compilación y runtime |
| Optimización de imágenes | La imagen final no contiene el compilador de Go ni el código fuente, solo el binario |
| Seguridad | Superficie de ataque mínima: Alpine base, un único binario, sin herramientas de build en producción |
| Deploy | Publicación a Vercel desde la CLI usando un `Dockerfile.vercel` dedicado |

## El Dockerfile: dos etapas, una responsabilidad cada una

```dockerfile
# ---------- Etapa 1: builder ----------
FROM golang:1.24-alpine AS builder

WORKDIR /src

COPY . .
RUN go build -o /server main.go

# ---------- Etapa 2: runner ----------
FROM alpine:3.20 AS runner
COPY --from=builder /server /server

CMD [ "/server" ]
```

### Etapa `builder`

Parte de `golang:1.24-alpine`, que trae todo el toolchain de Go (~250 MB). Su única misión es **compilar** `main.go` y producir un binario estático en `/server`. Todo lo que hay en esta etapa (compilador, caché de módulos, fuentes) se descarta al terminar.

### Etapa `runner`

Arranca desde cero con `alpine:3.20` (~8 MB) y con `COPY --from=builder` toma **únicamente el binario** compilado en la etapa anterior. Es la imagen que se ejecuta en producción.

### Por qué importa

| | Imagen single-stage (`golang:alpine`) | Imagen multi-stage (`alpine` + binario) |
| --- | --- | --- |
| Tamaño aproximado | ~250 MB+ | ~15 MB |
| Contiene compilador | Sí | No |
| Contiene código fuente | Sí | No |
| Tiempo de pull / cold start | Mayor | Mínimo |
| Superficie de ataque | Amplia | Reducida |

Nombrar las etapas (`AS builder`, `AS runner`) hace el Dockerfile legible y permite construir una etapa concreta cuando hace falta depurar:

```bash
docker build -f Dockerfile.vercel --target builder -t go-vercel:builder .
```

## La aplicación

```go
port := os.Getenv("PORT")
if port == "" {
    port = "80"
}
```

El servidor lee el puerto desde la variable de entorno `PORT`, que es la que inyecta la plataforma al arrancar el contenedor. Si no existe, usa `80` por defecto. Así la misma imagen funciona en local, en Vercel o en cualquier orquestador sin tocar código.

## Probarlo en local

```bash
# Construir la imagen final (etapa runner)
docker build -f Dockerfile.vercel -t go-vercel .

# Ejecutar mapeando el puerto
docker run --rm -p 8080:80 go-vercel

# Comprobar
curl http://localhost:8080
# Hello from a container on Vercel 👋
```

Para comparar tamaños entre etapas:

```bash
docker build -f Dockerfile.vercel --target builder -t go-vercel:builder .
docker images | grep go-vercel
```

## Deploy a Vercel desde la línea de comandos

Todo el despliegue se hace con la [Vercel CLI](https://vercel.com/docs/cli), sin repositorio conectado ni pipeline:

```bash
# 1. Instalar e iniciar sesión
npm i -g vercel
vercel login

# 2. Vincular la carpeta a un proyecto (crea .vercel/, ignorado en git)
vercel link

# 3. Desplegar (Vercel detecta Dockerfile.vercel y construye la imagen)
vercel

# 4. Promover a producción
vercel --prod
```

Vercel usa el archivo `Dockerfile.vercel` para construir la imagen, ejecuta la etapa final (`runner`) e inyecta `PORT` en el contenedor. El proyecto queda con el preset **Container**.

Resultado en producción:

```bash
curl https://05-vercel-bice.vercel.app
# Hello from a container on Vercel 👋
```

> La carpeta `.vercel/` contiene los IDs del proyecto y de la organización, por eso está en `.gitignore`.

## Estructura

```
.
├── main.go             # Servidor HTTP en Go
├── Dockerfile.vercel   # Build multi-stage: builder → runner
├── .gitignore          # Excluye .vercel/
└── README.md
```

## Lo que aprendí

1. Separar compilación y ejecución con **multi-stage builds** reduce drásticamente el tamaño y el riesgo de la imagen final.
2. `COPY --from=<etapa>` permite elegir con precisión qué artefactos pasan a producción.
3. Diseñar la app para leer su configuración del entorno (`PORT`) hace la imagen portable entre plataformas.
4. Vercel no es solo para frontends: puede ejecutar **contenedores Docker** y desplegarlos desde la terminal en segundos.
5. Un flujo de deploy sin CI es ideal para prototipos y pruebas rápidas, manteniendo fuera del repo la configuración local (`.vercel/`).

---

Hecho por [Rubén Rangel](https://github.com/rubenerangel) como parte de mi práctica avanzada con Docker.
