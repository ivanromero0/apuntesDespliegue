# Chuleta de examen: Virtualización y Docker

*Implantación de arquitecturas web y tecnologías de virtualización · basada en los 4 PDFs de clase (Docker 1, 2, 3 y Creación de imágenes)*

> **Esquema mental:** `Dockerfile` → (`docker build`) → **imagen** → (`docker run`) → **contenedor**. Los **volúmenes** persisten los datos y un **registry** (Docker Hub) sirve para compartir imágenes.

## 1. Virtualización: máquinas virtuales vs contenedores

**Virtualización** = emulación por software del funcionamiento de una infraestructura de servidores real, con un entorno de ejecución idéntico al físico. Responde a la necesidad de recursos «elásticos» que se adapten a procesos de cómputo cambiantes.

**Máquina virtual (VM):** un sistema operativo completo funcionando aislado sobre otro SO completo. Permite compartir el hardware entre varios SO a la vez. Necesita un **hipervisor**, un software que expone los recursos hardware a otros SO (Hyper-V, VirtualBox, VMware).

- Ventajas: reduce gastos (un recurso físico hace el trabajo de muchos); gestión centralizada de todos los entornos desde el hipervisor; está aislada del host, un lugar seguro para experimentar.
- Inconvenientes: ocupa muchos recursos (una copia de un SO y del hardware virtual por cada aplicación); reubicar una aplicación es complicado porque está atada a su SO; rara vez aprovecha todos los recursos asignados.

**Contenedor:** filosofía distinta. En lugar de un SO completo, **comparte los recursos (el kernel) del SO host**. Ejecuta una aplicación y sus servicios de forma aislada e independiente, buscando que se comporte igual en todos los entornos.

- Ventajas: instalación sencilla (una línea de comando); independiente de la plataforma (solo hace falta un SO que soporte contenedores); pérdidas por virtualización mínimas (sin hipervisor ni otros SO); aplicaciones aisladas entre sí.
- Inconvenientes: menos herméticos que una VM; menor seguridad y estabilidad porque comparten el núcleo del host.

|  | Máquina virtual | Docker (contenedor) |
| --- | --- | --- |
| Aislamiento | De procesos a nivel de **hardware** | De procesos a nivel de **SO** |
| Sistema operativo | Cada VM tiene uno independiente | Los contenedores pueden compartir SO |
| Arranque | En **minutos** | En **segundos** |
| Tamaño | Pocos **GB** | Livianos (**KB / MB**) |
| Disponibilidad | VMs listas para usar, difíciles de encontrar | Contenedores prediseñados fácilmente disponibles |
| Movilidad | Se mueven a un nuevo host fácilmente | Se **destruyen y recrean** en lugar de moverse |
| Creación | Lleva relativamente más tiempo | En segundos |
| Recursos | Más uso | Menor uso |

## 2. Docker: conceptos básicos

- Proyecto **open source** de 2013, creado por la empresa **dotCloud** (después **Docker Inc.**). Escrito en **Go**.
- Usa características del kernel de Linux (**espacios de nombres** y **grupos de control**) para crear contenedores sobre el SO y automatizar el despliegue de aplicaciones.
- **Arquitectura cliente-servidor:** el **cliente** de Docker (interfaz principal; acepta comandos del usuario) habla con el **daemon** (crea, ejecuta y distribuye los contenedores). Ambos pueden estar o no en la misma máquina.

| Término | Qué es |
| --- | --- |
| **Imagen** | Plantilla con la que se crean los contenedores; está compuesta por una serie de **capas**. |
| **Etiqueta (tag)** | Versión de una imagen (`nombre:tag`); si no se indica, se usa `latest`. |
| **Registry** | Repositorio de imágenes (Docker Hub). El daemon lo usa para descargar y subir imágenes. |
| **Contenedor** | Imagen en ejecución; se puede arrancar y parar como cualquier otro proceso. |
| **Docker Engine** | Tecnología que crea y gestiona contenedores: un servidor, una API y una interfaz de línea de comandos. |
| **Dockerfile** | Archivo de texto con instrucciones para construir una imagen (SO base, dependencias, configuración, comandos). |
| **Volumen** | Almacenamiento gestionado por Docker que persiste los datos aunque el contenedor se pare o elimine; se puede compartir entre contenedores. |

**Instalación en Ubuntu** (https://docs.docker.com/engine/install/ubuntu/):

```bash
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
#  apt-transport-https: permite al gestor de paquetes transferir datos por https
#  ca-certificates: permite verificar certificados de seguridad
#  curl: herramienta para transferir datos
#  software-properties-common: scripts para administrar software
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -     # clave GPG: verifica que el software es auténtico
sudo add-apt-repository 'deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable'   # repositorio de Docker
sudo apt-get update
sudo apt-get install docker-ce                      # última versión de Docker
```

**Primera prueba:** `docker run hello-world`. Por detrás: (1) el cliente conecta con el daemon; (2) el daemon descarga la imagen `hello-world` de Docker Hub; (3) crea un contenedor de esa imagen y ejecuta el proceso, que genera la salida; (4) el daemon redirige la salida al cliente, que la muestra en el terminal.

**Workflow típico:** construir una imagen desde un Dockerfile (o descargar una de un registry) → ejecutarla para generar un contenedor → arrancarlo y pararlo como cualquier proceso → si queremos compartir la imagen, publicarla en un registry.

## 3. Comandos sobre imágenes

En clase se usan con `sudo` (si tu usuario no está en el grupo `docker`).

```bash
sudo docker search nginx                 # busca imágenes en Docker Hub
sudo docker pull nginx:1.25              # descarga una imagen concreta (nombre:tag)
sudo docker images                       # lista las imágenes locales
docker image ls                          # equivalente a docker images
docker image ls -a                       # todas, también las huérfanas (--all)
docker image ls -q                       # solo los IDs (--quiet)
docker image ls -f dangling=true         # filtra por un valor (--filter)
docker image inspect nginx:1.25          # detalles: capas y otros elementos
docker image history nginx:1.25          # historia: las capas que forman la imagen
docker rmi nginx:1.25                    # elimina una imagen (por nombre o por ID)
docker rmi imagen1 imagen2               # varias a la vez
```

- `docker image` es el comando principal para gestionar imágenes: descargarlas, listarlas, eliminarlas, construirlas, subirlas a un registro y revisar su contenido.
- Una imagen **no se puede eliminar si hay un contenedor que la usa**.
- Una imagen **huérfana** es la que no ha terminado de descargarse, o la que se genera con `docker build` sin `-t` (sin etiqueta).

## 4. Contenedores: `docker run` y gestión

Sintaxis: `docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]`

| Opción | Qué hace |
| --- | --- |
| `-d` | **Detached**: en segundo plano, deja libre el prompt (servicios o tareas largas). |
| `-it` | **Interactivo**: mantiene el contenedor en primer plano y te permite interactuar con él. |
| `--name nombre` | Asigna nombre al contenedor (si no, Docker pone uno aleatorio). |
| `--rm` | Elimina el contenedor al terminar su ejecución. |
| `-p host:contenedor` | Publica un puerto: `[ip]:puerto_host:puerto_contenedor`. |
| `-P` | Redirige **todos** los puertos expuestos del contenedor (pierdes el control del puerto; útil para pruebas). |
| `-e VAR=valor` | Define una variable de entorno (puedes poner varias `-e`). |
| `-v ...` | Asocia un volumen (ver sección 5). |
| comando final | Comando a ejecutar al iniciar; **siempre es el último parámetro**, incluso detrás del nombre de la imagen. |

En algunas diapositivas aparecen `-n` y `-rm`: la forma correcta es `--name` y `--rm`.

```bash
docker run hello-world
docker run --name web -d -p 8080:80 nginx          # nginx en segundo plano; localhost:8080 -> puerto 80 del contenedor
docker run -it --rm --name prueba ubuntu bash      # shell interactiva; el contenedor se borra al salir
docker run -d -p 6379:6379 redis                   # puerto de mi máquina : puerto del contenedor
docker run -d -p 127.0.0.1:6379:6379 redis         # limitado a una IP de mi máquina
docker run -d -P nginx                             # publica todos los puertos expuestos en puertos aleatorios
docker run -e VAR1=valor1 -e VAR2=valor2 mi_imagen # variables de entorno
docker run ubuntu ifconfig                         # al iniciar ejecuta ifconfig en vez del comando por defecto
```

**Gestión de contenedores:**

```bash
docker ps                          # contenedores en ejecución
docker ps -a                       # todos, sin importar el estado
docker start nombre                # arranca (también: stop, restart, kill, pause, unpause)
docker stop nombre
docker logs nombre                 # logs del contenedor
docker inspect nombre              # información detallada del contenedor
docker rename nombre nuevo_nombre  # renombra
docker rm nombre                   # elimina un contenedor PARADO
docker rm -f nombre                # lo elimina sea cual sea su estado
```

**Acceder a un contenedor en ejecución:**

```bash
docker attach nombre               # te conecta al proceso principal (depuración o pasarle datos)
docker exec nombre ls /etc         # ejecuta un comando dentro de un contenedor en marcha
docker exec -it nombre bash        # abre una shell dentro del contenedor
```

**Crear una imagen desde un contenedor (`docker commit`):** guarda el estado actual del contenedor como imagen, incluyendo la imagen base y **todos los cambios hechos a mano** (datos, cambios directos, archivos copiados).

```bash
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
docker commit web mi_web:v1       # contenedor web -> imagen mi_web:v1
```

Ojo: en una diapositiva de repaso aparece con los argumentos al revés; el orden correcto es **primero el contenedor, después la imagen**.

## 5. Volúmenes

Un **volumen** es una carpeta del sistema de ficheros del host que **sobrevive al ciclo de vida del contenedor**. Es la forma de **persistir los datos**: se alojan fuera del contenedor, así que puedes cambiar, apagar o borrar el contenedor sin afectarlos.

- Docker **nunca elimina automáticamente** los volúmenes al borrar un contenedor ni «recoge basura» (volúmenes huérfanos).
- Los volúmenes son específicos de cada contenedor: de una misma imagen puedes crear n contenedores con volúmenes distintos.
- También sirven para **compartir información entre contenedores**.
- No puedes eliminar un volumen asociado a un contenedor.
- Se asocian a un contenedor con la opción `-v` de `docker run`.

```bash
docker volume ls                       # lista los volúmenes
docker volume create vol1              # crea un volumen (también: --name vol1)
docker volume inspect vol1             # detalles del volumen
docker volume rm vol1                  # lo elimina (no si lo usa un contenedor)
```

| Tipo | Qué es | Ejemplo |
| --- | --- | --- |
| **Anonymous data volume** | Carpeta creada en `/var/lib/docker/` al levantar un contenedor sin darle nombre. Docker cuenta los contenedores que lo usan y solo se elimina cuando se borra el último; si no, queda «parasitando» hasta `docker volume rm`. | `docker run -ti --rm -v /data go_example` |
| **Named data volume** | No depende de ningún contenedor y se puede montar en cualquiera. Se crea con `docker volume create` o al ejecutar un contenedor dándole nombre. Nunca se elimina solo. | `docker volume create --name vol1` y `docker run -ti -v vol1:/data go_example` |
| **Mounted volume** (bind) | Monta ficheros o carpetas del host indicando la ruta completa en el host y la ruta completa en el contenedor. | `docker run -ti --rm -v /etc/hostname:/root/parent_name -v /opt/:/data go_example` |

Formato de `-v`: `origen:destino`. Si el origen es un nombre, es un volumen nombrado; si es una ruta absoluta del host, es un volumen montado.

## 6. Dockerfile: instrucciones

`docker build` parte de un **Dockerfile** (texto plano con las instrucciones para crear la imagen) y un **contexto** (conjunto de archivos de la ruta, path o URL donde reside el Dockerfile). La sintaxis son comentarios (`#`) y `COMANDO argumentos`. Cada instrucción crea una **capa** de la imagen.

| Instrucción | Función y reglas | Ejemplo |
| --- | --- | --- |
| `FROM` | Imagen base. **Obligatoria y la primera** instrucción; probablemente la más importante. | `FROM ubuntu:latest` |
| `RUN` | Ejecuta comandos en la capa más alta y guarda el resultado. Formato shell (`RUN comando`) o exec (`RUN ["ejecutable", "param1"]`). Se concatena con `&&` y `\` continua la línea. | `RUN apt-get update && apt-get install nginx -y` |
| `COPY` | Copia ficheros **del contexto** a la imagen. Formatos: `COPY [--chown=u:g] <src>... <dest>` y la forma JSON (si hay espacios en las rutas). | `COPY index.html /var/www/html` |
| `ADD` | Igual que COPY pero admite **URL** como origen. Se recomienda usar COPY. | `ADD https://ejemplo.com/f.tar /tmp/` |
| `WORKDIR` | Directorio de trabajo para los `RUN`, `CMD`, `ENTRYPOINT`, `COPY` y `ADD` siguientes; si no existe, se crea. | `WORKDIR /var/www/html` |
| `EXPOSE` | Indica que el contenedor escucha en esos puertos (TCP por defecto, o `/udp`). **No los publica**: hay que usar `-p` en `docker run`. | `EXPOSE 80` |
| `VOLUME` | Crea un punto de montaje marcado como volumen externo. Admite array JSON o cadena simple. `docker run` lo inicializa con los datos que haya en esa ruta de la imagen y se crea un volumen anónimo. | `VOLUME /var/log` |
| `CMD` | Valores por defecto del contenedor: comando que se ejecuta al arrancar. **Solo cuenta el último CMD**. | `CMD ["nginx", "-g", "daemon off;"]` |
| `ENTRYPOINT` | Comando y parámetros que se ejecutan primero. Si no hay, se ejecuta `/bin/sh -c`. | `ENTRYPOINT ["/bin/echo"]` |
| `ENV` | Variable de entorno para las instrucciones posteriores. `ENV clave valor` (una) o `ENV clave=valor` (varias). | `ENV miVariable=2` |
| `LABEL` | Metadatos de la imagen: `LABEL clave=valor ...` | `LABEL version=1.0` |
| `USER` | Usuario (o UID) y grupo (o GID) para los `RUN`, `CMD` y `ENTRYPOINT` siguientes. | `USER antoni` |
| `ARG` | Variable que se pasa en el build con `--build-arg`; admite valor por defecto. | `ARG version=1.0` |

**Reglas de `COPY`:**

- `--chown` solo funciona si la imagen base es Linux.
- El origen admite comodines: `COPY hom* /mydir/`.
- El origen **no puede estar fuera del contexto**: `COPY ../something /algo` no está permitido.
- El destino es absoluto o relativo al `WORKDIR`: `COPY test relativeDir/` copia a `WORKDIR/relativeDir/`; `COPY test /absoluteDir/` copia a `/absoluteDir/`.
- Con varios orígenes (o comodín), el destino debe ser un directorio y terminar en `/`.
- Si el destino no termina en `/`, se trata como fichero; si no existe, se crea con todos sus directorios.

**`CMD` y `ENTRYPOINT`**

- Formatos de CMD: `CMD ["ejecutable", "param1"]` (forma preferida), `CMD ["param1", "param2"]` (parámetros por defecto de ENTRYPOINT) y `CMD param1 param2` (shell).
- Formatos de ENTRYPOINT: exec `["executable", "param"]` y shell `command param`.
- Solo CMD: se ejecuta con el ENTRYPOINT por defecto `/bin/sh -c`. Con los dos: ENTRYPOINT es el ejecutable y CMD **sus parámetros por defecto**.
- Los argumentos de `docker run imagen args` se **anexan** tras ENTRYPOINT y **anulan** el CMD.
- Para anular el ENTRYPOINT: `docker run --entrypoint otro_comando imagen`.

```dockerfile
FROM ubuntu:16.04
CMD ["/bin/date"]               # se ejecuta como: /bin/sh -c /bin/date
```

```dockerfile
FROM ubuntu:16.04
ENTRYPOINT ["/bin/echo"]        # ejecutable
CMD ["Hello"]                   # parámetro por defecto
```

```bash
docker run mi_echo                    # imprime: Hello
docker run mi_echo Adios              # imprime: Adios (el argumento anula el CMD)
docker run --entrypoint ls mi_echo -l # sustituye el ENTRYPOINT
```

## 7. `docker build`

Formato: `docker build [OPTIONS] PATH | URL | -`

| Opción | Qué hace |
| --- | --- |
| `-t`, `--tag` | Etiqueta la imagen (`nombre:tag`); se pueden aplicar varias. Sin ella, la imagen queda **huérfana**. |
| `-f`, `--file` | Usa un Dockerfile con otro nombre o ruta. |
| `--label` | Añade metadatos (mismo efecto que `LABEL`). |
| `--build-arg` | Da valor a una variable `ARG` en tiempo de construcción. |
| `--no-cache` | Por defecto Docker usa caché; con esta opción genera la imagen sin ella. |
| `--pull` | Fuerza descargar la nueva versión de la imagen del `FROM` (por defecto solo la descarga si no existe). |
| `--quiet` | Solo muestra el ID de la imagen creada. |
| `--target` | Construye solo hasta una etapa concreta (multi-stage). |

```bash
docker build -t mi_imagen:1.0 .                 # Dockerfile del directorio actual
docker build -f otro_dockerfile -t mi_imagen:1.0 .
docker build --label 'Author=antoni' -t mi_imagen .
docker build --build-arg some_variable_name=a_value -t mi_imagen .
docker build --no-cache --pull -t mi_imagen:1.0 .
docker build --target builder -t antoni/maven-stage .
```

```dockerfile
ARG some_variable_name                       # o con valor por defecto: ARG some_variable_name=default_value
RUN echo Oh dang look at that $some_variable_name
```

## 8. Multi-stage build

Permite usar **varios `FROM`** en un mismo Dockerfile; **la imagen final es la del último `FROM`**. Con `COPY --from=etapa` se copian archivos de una etapa anterior (o de una imagen del sistema). Así la imagen final no arrastra las herramientas de compilación.

```dockerfile
FROM maven:3.6.2-ibmjava-8 as builder
COPY app /app
RUN cd /app && mvn package

FROM openjdk:8-alpine
COPY --from=builder /app/target/ejecutable.jar /opt/ejecutable.jar
CMD ["java", "-jar", "/opt/ejecutable.jar"]
```

No hace falta construir todas las etapas: `docker build --target builder -t antoni/maven-stage .`

**Ejemplo de ahorro (Go):** sin multi-stage la imagen pesa lo de `golang:1.11` (**\~800 MB**); con multi-stage pesa lo de `debian:stretch` (**\~100 MB**).

```dockerfile
# Sin multi-stage (~800 MB)
FROM golang:1.11
WORKDIR /app
COPY main.go .
RUN go build -o server .
CMD ["./server"]
```

```dockerfile
# Con multi-stage (~100 MB)
FROM golang:1.11 as builder
WORKDIR /app
COPY main.go .
RUN go build -o server .

FROM debian:stretch
WORKDIR /app
COPY --from=builder /app .
CMD ["./server"]
```

## 9. Buenas prácticas en Dockerfiles

- **Usar `.dockerignore`** para que `docker build` excluya del contexto ficheros que no deben ir a la imagen:

```text
.git/
node_modules/
dist/
```

- **Fusionar comandos `RUN`:** cada instrucción crea una capa, así que agrupa las que tienen una probabilidad similar de cambiar.
- **Etiquetar la imagen base:** `latest` apunta a otra imagen cuando sale una versión nueva y el build puede romperse. Usa `FROM ubuntu:16.04`, no `FROM ubuntu`.
- **Eliminar archivos innecesarios tras cada `RUN`** (por ejemplo `rm -rf /var/lib/apt/lists/*`).
- **Añadir metadatos** con `LABEL` (p. ej. `LABEL maintainer=antoni@gmail.com`); programas externos a veces los exigen.
- Usar multi-stage build para reducir el tamaño final.

```dockerfile
# Mal                                          # Bien
FROM ubuntu                                    FROM ubuntu:16.04
ADD . /app                                     RUN apt-get update && apt-get install -y nodejs \
RUN apt-get update \                               && rm -rf /var/lib/apt/lists/*
    && apt-get install -y nodejs \             ADD . /app
    && cd /app \                               RUN cd /app && npm install
    && npm install                             CMD npm start
CMD npm start
```

## 10. Ejemplos completos

### 10.1 Servidor web nginx con Dockerfile propio

Estructura: una carpeta `miweb/` con `Dockerfile` e `index.html`.

```dockerfile
# miweb/Dockerfile
FROM ubuntu:16.04                      # imagen base con etiqueta concreta
LABEL autor=Antoni                     # metadatos
LABEL version=1.0
RUN apt-get update && apt-get install -y nginx \
    && rm -rf /var/lib/apt/lists/*     # instala y limpia en la misma capa
COPY index.html /var/www/html          # copia desde el contexto
ENV miVariable=2                       # variable de entorno
WORKDIR /var/www/html                  # directorio de trabajo
USER root                              # usuario que ejecuta las acciones
VOLUME /var/log/nginx                  # punto de montaje para los logs
EXPOSE 80                              # documenta el puerto (no lo publica)
CMD ["nginx", "-g", "daemon off;"]     # arranca nginx en primer plano
```

```bash
cd miweb
echo '<h1>Hola Docker</h1>' > index.html
docker build -t miweb:1.0 .                          # construye la imagen
docker images                                        # comprueba que existe
docker run -d --name web -p 8080:80 miweb:1.0        # contenedor en segundo plano
curl http://localhost:8080                           # comprueba la web
docker ps                                            # está en ejecución
docker logs web                                      # logs
docker exec -it web bash                             # entra dentro
docker commit web miweb:1.1                          # guarda el estado como imagen nueva
docker stop web && docker rm web                     # para y elimina el contenedor
docker rmi miweb:1.0 miweb:1.1                       # elimina las imágenes
```

### 10.2 Persistir datos con un volumen nombrado

```bash
docker volume create datos                                   # crea el volumen
docker run -d --name db -v datos:/data -p 6379:6379 redis    # lo monta en /data
docker rm -f db                                              # borramos el contenedor...
docker run -d --name db2 -v datos:/data redis                # ...y los datos siguen en el volumen
docker volume inspect datos                                  # dónde está guardado
docker rm -f db2 && docker volume rm datos                   # limpieza: primero contenedor, luego volumen
```

```bash
# Mounted volume: compartir una carpeta del host con el contenedor
docker run -d -p 8080:80 -v /home/usuario/web:/usr/share/nginx/html nginx
```

### 10.3 Variables de entorno, ARG y comando final

```dockerfile
# Dockerfile
FROM alpine:3.10
ARG saludo=Hola                        # valor en tiempo de build
ENV MENSAJE=$saludo                    # lo conserva como variable de entorno
ENTRYPOINT ["/bin/echo"]
CMD ["valor por defecto"]
```

```bash
docker build --build-arg saludo=Buenas -t eco:1 .
docker run --rm eco:1                       # imprime: valor por defecto
docker run --rm eco:1 otro texto            # imprime: otro texto
docker run --rm -e MENSAJE=Hey eco:1 ok     # -e sobrescribe la variable de entorno
docker run --rm --entrypoint env eco:1      # cambia el ENTRYPOINT: muestra las variables
```

## 11. Repaso final y errores típicos

| Si te piden... | Usa... |
| --- | --- |
| Ejecutar en segundo plano / interactivo | `docker run -d` / `docker run -it` |
| Publicar un puerto | `-p host:contenedor` (`EXPOSE` solo lo documenta) |
| Ver contenedores (incluidos parados) | `docker ps -a` |
| Borrar un contenedor en marcha | `docker rm -f` |
| Ejecutar un comando en un contenedor activo | `docker exec nombre comando` |
| Guardar un contenedor como imagen | `docker commit contenedor imagen:tag` |
| Construir una imagen | `docker build -t nombre:tag .` |
| Ver capas de una imagen | `docker image history` |
| Persistir datos | `-v volumen:/ruta` |
| Imagen final pequeña | Multi-stage build |

**Errores típicos**

- `EXPOSE` **no** hace accesible el puerto desde el host; hace falta `-p` en `docker run`.
- Solo cuenta el **último `CMD`**; los argumentos de `docker run` anulan CMD y se anexan al ENTRYPOINT.
- El comando final de `docker run` va **siempre al final**, detrás del nombre de la imagen.
- `docker rm` solo borra contenedores **parados**; usa `-f` para forzar.
- No se puede `docker rmi` una imagen usada por un contenedor, ni `docker volume rm` un volumen en uso.
- Los volúmenes **no se borran solos**: los anónimos quedan huérfanos hasta `docker volume rm`.
- `COPY` no puede salir del contexto (`../algo` no vale) y `FROM` debe ser la primera instrucción.
- Sin `-t`, `docker build` genera una imagen huérfana; con la caché activa un build repetido puede no regenerar nada (`--no-cache`).
- Evita `latest` en `FROM`: etiqueta la imagen base con una versión concreta.
- Las diapositivas tienen erratas: `-n`/`-rm` son `--name`/`--rm`, y en `docker commit` el contenedor va antes que la imagen.
