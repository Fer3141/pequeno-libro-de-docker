# Pequeño libro de Docker

Guía rápida de Docker: imágenes, contenedores, Dockerfile, volúmenes, redes y Docker Compose, explicados en español.

## Índice

- [¿Qué es Docker?](#qué-es-docker)
- [Instalación](#instalación)
- [Imágenes](#imágenes)
- [Contenedores](#contenedores)
- [Dockerfile](#dockerfile)
- [Volúmenes](#volúmenes)
- [Redes](#redes)
- [Docker Compose](#docker-compose)
- [Registro de imágenes (Docker Hub)](#registro-de-imágenes-docker-hub)
- [Logs y debugging](#logs-y-debugging)
- [Limpieza del sistema](#limpieza-del-sistema)
- [Buenas prácticas](#buenas-prácticas)

---

## ¿Qué es Docker?

Docker empaqueta una aplicación junto con todo lo que necesita para correr (código, dependencias, librerías del sistema, configuración) en una unidad llamada **contenedor**. A diferencia de una máquina virtual, un contenedor no incluye un sistema operativo completo: comparte el kernel del host, lo que lo hace mucho más liviano y rápido de arrancar (segundos, no minutos).

Tres conceptos que conviene tener claros desde el principio:

- **Imagen**: una plantilla de solo lectura (código + dependencias + instrucciones de arranque) a partir de la cual se crean contenedores. Se define con un `Dockerfile`.
- **Contenedor**: una instancia en ejecución de una imagen. Se pueden crear muchos contenedores a partir de la misma imagen, cada uno aislado del resto.
- **Registro (registry)**: un repositorio de imágenes, público (Docker Hub) o privado, desde donde se descargan (`pull`) o donde se suben (`push`).

---

## Instalación

Se descarga desde la página oficial: https://www.docker.com/products/docker-desktop/ (incluye Docker Engine, CLI y Docker Compose).

- **Windows / macOS**: instalar **Docker Desktop** desde el enlace de arriba.
- **Linux**: instalar Docker Engine siguiendo la guía oficial según distribución: https://docs.docker.com/engine/install/

```bash
docker --version
docker info
```
`--version` confirma que la instalación funcionó. `docker info` muestra detalles del entorno: cuántos contenedores/imágenes hay, el driver de almacenamiento, si está corriendo en modo Linux o Windows containers, etc.

```bash
docker run hello-world
```
Descarga (si hace falta) y ejecuta una imagen mínima de prueba que solo imprime un mensaje de confirmación. Es el "hola mundo" para validar que Docker funciona de punta a punta (descarga de imagen + creación de contenedor + ejecución).

---

## Imágenes

```bash
docker images
```
Lista las imágenes descargadas localmente, con su repositorio, tag, ID y tamaño.

```bash
docker pull nginx:1.27
```
Descarga una imagen desde un registro (Docker Hub por defecto). El texto después de `:` es el **tag** (versión); si se omite, Docker usa `latest`, que en producción conviene evitar porque no es reproducible (`latest` puede apuntar a versiones distintas con el tiempo).

```bash
docker build -t mi-app:1.0 .
```
Construye una imagen a partir de un `Dockerfile` en el directorio actual (`.` es el contexto de build). `-t` le pone nombre y tag.

```bash
docker rmi mi-app:1.0
```
Elimina una imagen local. Falla si hay contenedores (incluso detenidos) que la usan; hay que eliminarlos primero o forzar con `-f`.

```bash
docker tag mi-app:1.0 usuario/mi-app:1.0
```
Crea un alias de la imagen con otro nombre, típicamente para prepararla antes de subirla a un registro bajo el namespace de usuario/organización.

---

## Contenedores

```bash
docker run -d --name mi-nginx -p 8080:80 nginx:1.27
```
Crea y arranca un contenedor a partir de una imagen. `-d` lo corre en segundo plano (detached). `--name` le da un nombre fácil de referenciar. `-p 8080:80` mapea el puerto 8080 del host al puerto 80 del contenedor (`host:contenedor`).

```bash
docker ps
docker ps -a
```
`docker ps` lista los contenedores en ejecución. `-a` incluye también los detenidos.

```bash
docker stop mi-nginx
docker start mi-nginx
docker restart mi-nginx
```
`stop` detiene el contenedor (le manda `SIGTERM`, y si no responde a tiempo, `SIGKILL`). `start` vuelve a arrancarlo conservando su estado. `restart` hace ambas cosas seguidas.

```bash
docker rm mi-nginx
docker rm -f mi-nginx
```
Elimina un contenedor detenido. `-f` lo detiene y elimina en un solo paso.

```bash
docker exec -it mi-nginx bash
```
Abre una terminal interactiva **dentro** de un contenedor ya corriendo. Es el comando más usado para inspeccionar o debuggear algo en vivo. Si la imagen no tiene `bash`, se prueba con `sh`.

```bash
docker inspect mi-nginx
```
Vuelca toda la configuración y el estado del contenedor en JSON: IP interna, variables de entorno, volúmenes montados, política de reinicio, etc.

```bash
docker cp mi-nginx:/etc/nginx/nginx.conf ./nginx.conf
```
Copia un archivo entre el host y el contenedor (funciona en ambos sentidos según el orden origen/destino).

> Un contenedor vive mientras vive su **proceso principal** (el definido por `CMD`/`ENTRYPOINT`). Si ese proceso termina, el contenedor se detiene, aunque no lo hayas parado explícitamente.

---

## Dockerfile

Un `Dockerfile` es la receta con la que se construye una imagen, instrucción por instrucción.

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "index.js"]
```

- `FROM` define la imagen base sobre la que se construye (siempre la primera instrucción). `node:20-alpine` usa una variante *alpine*, mucho más liviana que la imagen completa.
- `WORKDIR` fija el directorio de trabajo dentro de la imagen para las instrucciones siguientes (equivalente a un `cd`).
- `COPY origen destino` copia archivos del host a la imagen. Copiar primero solo `package*.json` y recién después el resto del código es una técnica deliberada: Docker cachea cada capa, así que si el código cambia pero las dependencias no, `npm install` no se vuelve a ejecutar.
- `RUN` ejecuta un comando **durante el build** y deja su resultado grabado en una nueva capa de la imagen.
- `EXPOSE` documenta qué puerto usa la aplicación dentro del contenedor (no lo publica por sí solo; eso lo hace `-p` en `docker run`).
- `CMD` define el comando que se ejecuta cuando arranca un contenedor a partir de la imagen. Solo puede haber uno (el último `CMD` gana si hay varios).

```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
```
`ENTRYPOINT` fija el proceso que siempre se ejecuta; `CMD` en este caso pasa a ser sus argumentos por defecto (que sí se pueden sobreescribir al hacer `docker run imagen otro-argumento`).

```dockerfile
ARG NODE_ENV=production
ENV NODE_ENV=$NODE_ENV
```
`ARG` define una variable solo disponible **durante el build** (se pasa con `docker build --build-arg NODE_ENV=development`). `ENV` define una variable de entorno que persiste **dentro del contenedor** en tiempo de ejecución.

**Multi-stage build**, para no cargar en la imagen final las herramientas usadas solo para compilar:

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN go build -o app .

FROM alpine:3.20
COPY --from=build /src/app /app
CMD ["/app"]
```
La primera etapa (`build`) compila el binario con todo el toolchain de Go. La segunda etapa parte de una imagen mínima (`alpine`) y solo copia el binario ya compilado con `COPY --from=build`. El resultado final no incluye el compilador ni el código fuente, así que pesa mucho menos.

---

## Volúmenes

Los contenedores son **efímeros**: si se borra el contenedor, se pierde todo lo que escribió en su sistema de archivos. Los volúmenes resuelven eso, persistiendo datos fuera del ciclo de vida del contenedor.

```bash
docker volume create datos-db
docker volume ls
docker volume inspect datos-db
```
`create` crea un volumen gestionado por Docker. `ls` los lista. `inspect` muestra dónde vive físicamente en el host.

```bash
docker run -d --name postgres -v datos-db:/var/lib/postgresql/data postgres:16
```
Monta el volumen `datos-db` en la ruta `/var/lib/postgresql/data` dentro del contenedor. Aunque el contenedor se borre, los datos sobreviven en el volumen y se pueden volver a montar en un contenedor nuevo.

```bash
docker run -d -v "$(pwd)/config:/etc/app/config" mi-app
```
**Bind mount**: monta una carpeta específica del host (no gestionada por Docker) dentro del contenedor. Útil en desarrollo para editar código en el host y verlo reflejado en vivo dentro del contenedor.

```bash
docker volume rm datos-db
docker volume prune
```
`rm` elimina un volumen puntual (falla si está en uso). `prune` elimina todos los volúmenes que no están siendo usados por ningún contenedor.

---

## Redes

Por defecto, cada contenedor tiene su propia IP interna y puede comunicarse con otros contenedores en la misma red por su **nombre** (Docker resuelve DNS interno automáticamente).

```bash
docker network ls
docker network create mi-red
```
`ls` lista las redes existentes (`bridge`, `host` y `none` vienen por defecto). `create` crea una red *bridge* propia: la forma recomendada de conectar contenedores entre sí, en vez de usar la red `bridge` por defecto.

```bash
docker run -d --name db --network mi-red postgres:16
docker run -d --name api --network mi-red -e DB_HOST=db mi-app
```
Ambos contenedores quedan en `mi-red`. Desde `api`, el hostname `db` resuelve automáticamente a la IP del contenedor `db`, sin necesidad de hardcodear IPs.

```bash
docker network inspect mi-red
```
Muestra qué contenedores están conectados a la red y sus IPs internas.

---

## Docker Compose

Compose describe, en un único archivo YAML, varios contenedores que trabajan juntos (una app + su base de datos + un caché, por ejemplo) y sus relaciones.

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DB_HOST=db
    depends_on:
      - db
  db:
    image: postgres:16
    volumes:
      - datos-db:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=secreto

volumes:
  datos-db:
```
Cada entrada bajo `services` es un contenedor. `build: .` construye la imagen desde el `Dockerfile` local; `image:` usa una ya publicada. `depends_on` controla el orden de arranque (no espera a que el servicio esté realmente "listo", solo a que el contenedor haya arrancado). `volumes` a nivel raíz declara volúmenes nombrados reutilizables entre servicios.

```bash
docker compose up -d
docker compose down
```
`up -d` crea y arranca todos los servicios definidos, en segundo plano (crea también la red interna que los conecta por nombre). `down` los detiene y elimina, junto con la red creada (los volúmenes nombrados se conservan salvo que se agregue `-v`).

```bash
docker compose ps
docker compose logs -f api
```
`ps` lista el estado de los servicios del proyecto. `logs -f <servicio>` sigue los logs de un servicio puntual en tiempo real.

```bash
docker compose build
docker compose restart api
```
`build` reconstruye las imágenes definidas con `build:` (por ejemplo, después de cambiar el `Dockerfile`). `restart` reinicia un servicio puntual sin tocar los demás.

---

## Registro de imágenes (Docker Hub)

```bash
docker login
```
Autentica la CLI contra Docker Hub (u otro registro con `docker login <registro>`).

```bash
docker tag mi-app:1.0 usuario/mi-app:1.0
docker push usuario/mi-app:1.0
```
Antes de subir una imagen hay que etiquetarla con el namespace del usuario/organización de destino. `push` la sube al registro.

```bash
docker pull usuario/mi-app:1.0
```
Descarga la imagen desde el registro a cualquier máquina con Docker instalado.

---

## Logs y debugging

```bash
docker logs mi-nginx
docker logs -f mi-nginx
docker logs --tail 100 mi-nginx
```
`logs` muestra la salida estándar/error del proceso principal del contenedor. `-f` sigue el stream en vivo (como `tail -f`). `--tail 100` limita a las últimas 100 líneas.

```bash
docker stats
```
Muestra en vivo el consumo de CPU, memoria, red y disco de todos los contenedores corriendo: el primer lugar donde mirar si algo está consumiendo más recursos de lo esperado.

```bash
docker top mi-nginx
```
Lista los procesos corriendo dentro del contenedor, como un `ps` a nivel del host pero acotado a ese contenedor.

```bash
docker events
```
Transmite en vivo los eventos del daemon de Docker (contenedores que arrancan, se detienen, mueren, imágenes que se construyen...). Útil para entender qué pasó justo antes de que algo fallara.

---

## Limpieza del sistema

```bash
docker system df
```
Muestra cuánto espacio en disco ocupan imágenes, contenedores, volúmenes y la caché de build.

```bash
docker container prune
docker image prune
docker image prune -a
```
`container prune` borra todos los contenedores detenidos. `image prune` borra imágenes "dangling" (sin tag, resultado de builds intermedios). `image prune -a` borra además todas las imágenes que no están en uso por ningún contenedor.

```bash
docker system prune -a --volumes
```
Limpieza total: contenedores detenidos, redes no usadas, imágenes sin uso y volúmenes no usados. **Cuidado**: `--volumes` borra también los datos persistidos que no estén en uso, y no es reversible.

---

## Buenas prácticas

- **Un proceso principal por contenedor**: no meter una app + su base de datos + un cron dentro del mismo contenedor. Cada servicio en su propio contenedor, orquestados con Compose (o Kubernetes), de forma que se puedan escalar, actualizar y reiniciar de forma independiente.
- **Evitar `latest` en producción**: fijar versiones explícitas (`nginx:1.27`, no `nginx:latest`) para que un despliegue sea reproducible y no cambie solo porque alguien subió una imagen nueva con ese tag.
- **Usar imágenes base livianas** (`-alpine`, `-slim`) cuando sea posible: menos superficie de ataque y builds/despliegues más rápidos.
- **Aprovechar la caché de capas**: ordenar el `Dockerfile` de lo que cambia menos (dependencias) a lo que cambia más (código de la app), para no reinstalar dependencias en cada build.
- **No correr como root dentro del contenedor** si no hace falta: agregar un usuario propio en el `Dockerfile` con `USER` reduce el impacto si alguien logra ejecutar código dentro del contenedor.
- **No guardar secretos en la imagen** (contraseñas, API keys): pasarlos como variables de entorno en tiempo de ejecución o con un gestor de secretos, nunca hardcodeados en un `ENV` del `Dockerfile`, porque quedan grabados en el historial de capas de la imagen.
- **Usar `.dockerignore`**: evita copiar `node_modules`, `.git` u otros archivos innecesarios al contexto de build, acelerando el build y reduciendo el tamaño de la imagen.
- **Healthchecks**: definir `HEALTHCHECK` en el `Dockerfile` (o `healthcheck:` en Compose) para que Docker sepa distinguir un contenedor "corriendo" de uno "corriendo pero roto".
