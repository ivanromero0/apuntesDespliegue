# Chuleta de examen: Git, GitHub Pages y GitHub Actions

*Despliegue de aplicaciones web · 2 DAW · basada en los 5 PDFs de clase (Resumen Git 1, 2 y 5; GitHub Actions 1 y 2)*

> **Regla de oro de Actions:** los workflows viven en `.github/workflows/nombre.yml` y siempre siguen `name → on → jobs → runs-on → steps`.

## 1. Conceptos de control de versiones

Un **SCV** registra los cambios sobre los archivos a lo largo del tiempo para poder recuperar versiones. Como mínimo debe ofrecer: almacenamiento de los elementos, posibilidad de cambiarlos y un registro histórico de las acciones.

**Para qué sirve:** marcha atrás y corrección de errores, compartir código, ver quién cambió qué y por qué (comentarios), y crear ramas que luego se mezclan con la principal.

| Tipo | Cómo funciona | Pros / contras | Ejemplos |
| --- | --- | --- | --- |
| Local | Copiar archivos a otro directorio (con fecha/hora) o guardar versiones en una BD local | Muy simple, pero muy propenso a errores | RCS |
| Centralizado | Un único servidor con todo el versionado; los clientes descargan de ahí | Todos saben en qué trabajan los demás y es fácil de administrar; **punto único de fallo** | CVS (años 80, primer SCV open source; solo acepta ficheros en la última versión), SVN (2000: commits atómicos, ramas eficientes, binarios como binarios, envía incrementos) |
| Distribuido | Cada desarrollador tiene una copia completa con toda la historia; se sincronizan entre sí | Cualquier clon puede restaurar el servidor | **Git**, Mercurial, Bazaar, BitKeeper |

**Fundamentos de Git**

- Guarda **instantáneas (snapshots)**, no copias completas; si un fichero no cambia, guarda un puntero al anterior.
- La mayoría de operaciones son **locales**.
- **Integridad:** no guarda por nombre de archivo sino por el **hash de su contenido**.
- **3 estados:** modificado (*modified*), preparado (*staged*), confirmado (*committed*).
- **3 áreas:** Working Directory, Staging Area, directorio `.git` (repositorio).

```text
working directory --git add--> staging area --git commit--> repo local --git push--> repo remoto
working directory <--------------git checkout / git merge---- repo local <--git pull---- repo remoto
```

## 2. Comandos básicos (repositorio local)

```bash
git init                        # crea la carpeta .git (repositorio local)
git init --bare                 # repo sin directorio de trabajo: solo como servidor
git status                      # qué está modificado, preparado o sin rastrear

git add README.md               # prepara un fichero concreto (admite patrones)
git add .                       # prepara todo (no es la mejor práctica)
git add -u                      # solo añade ficheros que ya están rastreados
git add -n .                    # simula lo que haría, sin cambiar nada

git commit                      # abre el editor por defecto para el mensaje
git commit -m 'Añade login'     # commit con mensaje
git commit --amend              # añade cambios al último commit (y edita el mensaje)
git commit --amend --no-edit    # igual, pero sin cambiar el mensaje
git commit -am 'Arregla bug'    # add + commit en un paso (solo ficheros ya rastreados)
```

**Diferencias (`git diff`)**

```bash
git diff                        # directorio de trabajo vs área de preparación
git diff --staged               # área de preparación vs último commit (igual: --cached)
git diff rama1 rama2            # diferencias entre dos ramas
```

**Historial (`git log`)** muestra id del commit, autor, fecha y mensaje.

```bash
git log                         # todos los commits, del más reciente al más antiguo
git log -n 3                    # los últimos 3
git log --oneline               # una línea por commit (hash corto + mensaje)
```

**Eliminar y deshacer**

```bash
git rm fichero.txt              # lo borra del directorio de trabajo y lo prepara (sirve para renombrar)
git rm --cached fichero.txt     # deja de rastrearlo, pero lo conserva en disco
git rm -f fichero.txt           # fuerza el borrado (área de preparación y de trabajo)

git reset fichero.txt           # saca el fichero del área de preparación (deja de rastrearlo)
git reset --soft HEAD~1         # elimina el último commit, cambios se mantienen en staging
git reset --hard HEAD~1         # elimina el último commit y PIERDE los cambios
git revert <hash>               # crea un commit nuevo que invierte el indicado
```

**`reset` vs `revert`:** `reset` mueve HEAD y la rama hacia atrás (reescribe el historial); `revert` no mueve HEAD, añade un commit inverso y es **más seguro** (ideal para commits ya publicados).

**`.gitignore`**: lista de patrones que Git no debe rastrear.

```gitignore
# ignora los archivos terminados en .a
*.a
# pero no lib.a
!lib.a
# ignora solo el TODO de la raíz, no subdir/TODO
/TODO
# ignora todo el directorio build/
build/
# ignora doc/notes.txt pero no doc/server/arch.txt
doc/*.txt
# ignora todos los .txt bajo doc/
doc/**/*.txt
```

Qué meter en él: credenciales y claves de API (inyectarlas por variable de entorno), `/.vscode`, logs, `.DS_Store`, `/dist` y `/build`, `/node_modules`, `/coverage`. Plantillas: https://github.com/github/gitignore

## 3. Repositorios remotos

```bash
git clone https://github.com/Netflix/netflix.github.com.git              # copia remoto en local
git clone https://github.com/Netflix/netflix.github.com.git proyectoNetflix  # carpeta destino con otro nombre

git remote add origin https://github.com/usuario/repo.git   # añade un remoto
git remote -v                   # lista los remotos
git remote show origin          # configuración detallada de un remoto

git push origin main            # envía los commits de la rama main al remoto origin
git push -u origin main         # igual y asocia la rama: los siguientes bastan con git push
git pull origin main            # descarga Y fusiona los cambios del remoto
git fetch origin                # descarga, pero NO fusiona (los deja en una rama oculta)
```

- Al clonar se crea automáticamente el remoto `origin`.
- `git push` **no** envía las etiquetas.
- Haz **commit antes de hacer pull**.
- `pull` = `fetch` + `merge`.

**Crear un repo nuevo y subirlo a GitHub (el que muestra GitHub):**

```bash
echo '# ejemploRepo' >> README.md
git init
git add README.md
git commit -m 'first commit'
git branch -M main
git remote add origin https://github.com/dawEstacio/ejemploRepo.git
git push -u origin main
```

**Subir un repo local ya existente:**

```bash
git remote add origin https://github.com/dawEstacio/ejemploRepo.git
git branch -M main
git push -u origin main
```

## 4. Ramas

Una rama es una bifurcación del código para trabajar en paralelo. Son solo **punteros a commits**: crear una rama solo crea un puntero nuevo. Crear una rama **no te cambia a ella**: hay que hacer `checkout`.

```bash
git branch nueva-rama           # crea la rama (no cambia a ella)
git branch -a                   # lista todas (locales y remotas)
git branch -r                   # solo las remotas
git branch -vv                  # cada rama con su rama remota asociada

git checkout nueva-rama         # cambia a la rama (actualiza el directorio de trabajo)
git checkout -b nueva-rama      # crea la rama Y cambia a ella

git push -u origin nueva-rama   # publica la rama en el remoto
git branch -d nueva-rama        # borra la rama local
git push origin --delete nueva-rama   # borra la rama en el remoto

git checkout main               # 1) ponte en la rama RECEPTORA
git pull                        # 2) actualízala (y la que vas a fusionar)
git merge nueva-rama            # 3) fusiona nueva-rama dentro de main
```

- Al clonar solo obtienes el directorio de trabajo de la rama por defecto. Para traer otra rama remota: `git checkout nombre_rama` con el mismo nombre.
- `git merge` integra líneas de desarrollo en una sola rama y crea un commit de fusión.

## 5. Etiquetas (tags)

Marcan puntos importantes del historial (normalmente versiones). Hay dos tipos: **ligeras** y **anotadas** (llevan mensaje).

```bash
git tag                                  # lista etiquetas (orden alfabético)
git tag -l 'v1.*'                        # busca con un patrón
git tag v1.0                             # crea etiqueta ligera
git tag -a v1.0 -m 'Primera versión'     # crea etiqueta anotada
git log --oneline                        # para localizar el hash de un commit antiguo
git tag -a v0.9 -m 'Beta' 6413584        # etiquetado tardío (sobre un commit anterior)

git push origin v1.0                     # publica UNA etiqueta
git push origin --tags                   # publica TODAS las etiquetas

git tag -d v1.0                          # borra la etiqueta en local
git push origin :refs/tags/v1.0          # borra la etiqueta en el remoto

git checkout -b rama-v1 v1.0             # no se puede hacer checkout directo a un tag: crea una rama desde él
```

## 6. GitHub Pages, fork y pull request

**GitHub Pages** publica una web de tu proyecto. El contenido puede ser un `README.md` o ficheros `.html`. Lo habitual es usar una rama `gh-pages` solo con la documentación/web (aunque puede ser otra rama).

1. Crear el repositorio.
2. Crear el README o la estructura HTML y subirla al remoto (a `gh-pages` o a otra rama).
3. En **Settings → Pages** elegir la rama (y la carpeta, `/ (root)`) y, opcionalmente, un **tema** (solo se aplica si el contenido es un README.md). Se puede configurar *Custom domain* y *Enforce HTTPS*.
4. La web queda en `https://usuario.github.io/repositorio/`.

```bash
git checkout -b gh-pages
git add index.html
git commit -m 'Añade la web'
git push -u origin gh-pages
```

**Fork** = copiar un repositorio a tu cuenta para experimentar sin afectar al original. Luego se clona y se trabaja normal.

```bash
git clone https://github.com/miUsuario/gitFlow.git
cd gitFlow
git remote add upstream https://github.com/dawEstacio/gitFlow.git   # el original (el nombre upstream es solo convención)
git fetch upstream                  # trae los cambios del original
git checkout master
git merge upstream/master           # los fusiona en tu copia
```

**Pull request (PR):** propones al dueño del repo original que incluya tus cambios.

1. Haces cambios en una rama de tu fork y los subes (`git push`).
2. En el repo original: **New pull request** (opción *compare across forks*).
3. **base fork** = rama `master` del original; **head fork** = tu rama.
4. **Create pull request**: título, descripción, revisores (*Reviewers*), etc.
5. El mantenedor revisa y pulsa **Merge pull request**; el PR queda marcado como **merged**.

## 7. Buenas prácticas con Git

- Acordar una **metodología de ramas** (master/main, develop, feature, release, hotfix) y cuándo crear nuevas.
- **Mensajes de commit:** asunto separado del cuerpo por una línea en blanco; asunto de **máx. 50 caracteres**, empieza en mayúscula, **sin punto final** y en **imperativo**; cuerpo a **máx. 72 caracteres por línea**, explicando qué se resolvió y cómo.
- **Añadir solo los archivos con los que has trabajado** (`git add documento.txt`) en vez de `git add .`, para no colar archivos basura.
- **No subir commits con código incompleto**; comparte código completo.
- **Un commit por funcionalidad o requerimiento**, sin mezclar ficheros no relacionados: así se puede hacer `revert` sin problemas.

## 8. CI/CD

- **Continuous Integration (CI):** integrar código en el repositorio compartido varias veces al día; cada integración se verifica con **tests automatizados** para detectar errores pronto.
- **Continuous Delivery (CD):** extensión de CI; el código siempre está **listo para producción** tras pasar pruebas y validaciones, pero el despliegue **no es automático** (lo decide una persona).
- **Continuous Deployment (CD):** cada cambio aprobado y verificado se **despliega a producción automáticamente, sin intervención humana**; exige mucha confianza en los tests.

```text
Codificar -> Build -> Testear -> Crear release -> Desplegar
|--- Continuous Integration ---|
|-------- Continuous Delivery ---------|
|------------ Continuous Deployment -------------|
```

## 9. GitHub Actions: conceptos

Herramienta de GitHub para automatizar flujos CI/CD (https://github.com/features/actions).

| Componente | Qué es |
| --- | --- |
| **Workflow** | Procedimiento automatizado de un repositorio; tiene uno o más jobs y se activa por un **evento** (push, PR...). |
| **Job** | Conjunto de steps que se ejecutan en el **mismo runner**. Por defecto los jobs corren **en paralelo**. |
| **Step** | Tarea individual dentro de un job: una **action** (`uses`) o un **comando shell** (`run`). |
| **Action** | Comando independiente y reutilizable; el bloque portátil más pequeño. Pueden ser de GitHub, de terceros (Marketplace) o propias. |
| **Runner** | Servidor con la aplicación de ejecución de Actions instalada; ejecuta los jobs y devuelve el resultado. |

Jerarquía: **Workflow → Job → Step → Action**.

|  | Runner *GitHub-hosted* | Runner *self-hosted* |
| --- | --- | --- |
| Dónde | En la nube de GitHub | En tu propia máquina/red |
| Mantenimiento | Cero | Lo configuras y mantienes tú |
| Coste | Se paga por uso (gratis: 2000 min/mes) | No se paga por uso |
| SO | Windows, macOS y Ubuntu | El que tú quieras |
| Flexibilidad | Limitada para configuraciones muy específicas | Total; accede a tu red local y recursos propios |

**Elementos principales de un workflow** (fichero YAML en `.github/workflows/`):

| Clave | Requerido | Función |
| --- | --- | --- |
| `name` | No | Nombre en la pestaña Actions (si falta, usa la ruta del fichero). |
| `on` | **Sí** | Evento(s) que lanzan el workflow: cadena, lista, lista de tipos o mapa de configuración. |
| `jobs` | **Sí** | Agrupa los jobs; cada job necesita un identificador. |
| `jobs.<id>.runs-on` | **Sí** | Máquina donde corre el job (`ubuntu-latest`, `windows-latest`, `macos-latest` o `self-hosted`). |
| `jobs.<id>.needs` | No | Jobs que deben terminar antes (evita el paralelismo). Si uno falla, sus sucesores no se ejecutan. |
| `env` / `jobs.<id>.env` | No | Variables de entorno; si se define a varios niveles, gana la **más específica**. |
| `jobs.<id>.steps` | **Sí** | Lista de steps, se ejecutan **en secuencia**. |
| `steps.uses` | — | Ejecuta una action (`actions/checkout@v4`, o propia: `./.github/actions/mi-action`). |
| `steps.run` | — | Ejecuta un comando de shell (`run: \|` para varios). `working-directory` cambia la carpeta. |
| `steps.with` | No | Parámetros de entrada de una action. |
| `strategy.matrix` | No | Crea varios jobs sustituyendo variables (SO, versiones...). |

Sintaxis completa: https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions

## 10. Ejemplos completos de workflows (.yml)

Todos van en `.github/workflows/` (extensión `.yml` o `.yaml`, indentación con **espacios**, nunca tabuladores). La ruta de cada fichero aparece en la primera línea.

### 10.1 Primer workflow (el de clase)

```yaml
# .github/workflows/primer-ejemplo.yml
name: primer-ejemplo            # opcional: nombre que sale en la pestaña Actions
on: [push]                      # evento que lo dispara (requerido)
jobs:
  check-bats-version:           # id del job
    runs-on: ubuntu-latest      # runner de GitHub con Ubuntu (requerido)
    steps:                      # se ejecutan en secuencia
      - uses: actions/checkout@v4       # uses = usar una action (descarga el código)
      - uses: actions/setup-node@v4     # instala Node
      - run: npm install -g bats        # run = comando de shell
      - run: bats -v
```

Las diapositivas usan `@v1` / `@v2`; hoy se usan versiones más recientes (`@v4`), pero la estructura es idéntica.

### 10.2 Eventos (`on`)

```yaml
# .github/workflows/eventos.yml
name: eventos
on:
  push:
    branches:
      - main
    tags:
      - v1                 # push de la etiqueta v1
  pull_request:
    branches:
      - main
  workflow_dispatch:       # lanzamiento manual desde la pestaña Actions
  schedule:
    - cron: '0 8 * * 1'    # lunes a las 8:00 UTC
jobs:
  saludo:
    runs-on: ubuntu-latest
    steps:
      - run: echo Evento ${{ github.event_name }}
```

Formas cortas: `on: push`, `on: [push, pull_request]`.

### 10.3 Jobs dependientes (`needs`) y `always()`

```yaml
# .github/workflows/needs.yml
name: needs-ejemplo
on: [push]
jobs:
  job1:
    runs-on: ubuntu-latest
    steps:
      - run: echo Primero
  job2:
    needs: job1                 # espera a que job1 termine bien
    runs-on: ubuntu-latest
    steps:
      - run: echo Segundo
  job3:
    needs: [job1, job2]         # espera a los dos
    if: always()                # se ejecuta aunque job1 o job2 fallen
    runs-on: ubuntu-latest
    steps:
      - run: echo Tercero
```

Sin `needs`, los jobs corren **en paralelo**. Si un job falla, sus sucesores no se ejecutan (salvo con `always()`).

### 10.4 Variables de entorno a tres niveles

```yaml
# .github/workflows/env.yml
name: env-ejemplo
on: [push]
env:                            # nivel workflow
  SERVER: production
jobs:
  job1:
    runs-on: ubuntu-latest
    env:                        # nivel job
      FIRST_NAME: Mona
    steps:
      - name: Print a greeting
        env:                    # nivel step (si se repite, gana el más específico)
          MY_VAR: Hi there! My name is
          LAST_NAME: Octocat
        run: |
          echo $MY_VAR $FIRST_NAME $LAST_NAME
          echo Servidor: $SERVER
```

### 10.5 Matrix multi-SO con Python

```yaml
# .github/workflows/python-matrix.yml
name: Python CI multi-SO
on:
  push:
    branches: [main]
  pull_request:
jobs:
  test:
    runs-on: ${{ matrix.os }}              # cada combinación usa su propio SO
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.10', '3.11', '3.12']
    steps:
      - name: Descargar código
        uses: actions/checkout@v4
      - name: Instalar Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:                               # with = parámetros de entrada de la action
          python-version: ${{ matrix.python-version }}
      - name: Instalar dependencias
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      - name: Ejecutar tests
        run: python -m pytest
```

Genera 3 × 3 = **9 jobs** (uno por combinación).

### 10.6 Matrix con `include` (variables asociadas a cada versión)

```yaml
# .github/workflows/node-include.yml
name: Node.js CI
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - node-version: 10.x
            site: prod
            datacenter: site-a
          - node-version: 12.x
            site: dev
            datacenter: site-b
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
      - name: Echo site details
        env:
          SITE: ${{ matrix.site }}
          DATACENTER: ${{ matrix.datacenter }}
        run: echo $SITE $DATACENTER
```

La matrix sencilla de clase combinaba `os: [ubuntu-16.04, ubuntu-18.04]` con `node: [6, 8, 10]` y usaba `runs-on: ${{ matrix.os }}`.

### 10.7 CI + CD: tests y push a un Docker registry

```yaml
# .github/workflows/ci-cd.yml
name: CI-CD
on:
  push:
    branches: [main]
jobs:
  ci:                                   # Job CI
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4       # descargar código fuente
      - uses: actions/setup-node@v4     # preparar node
        with:
          node-version: 20
      - run: npm ci                     # instalar dependencias
      - run: npm test                   # tests
  cd:                                   # Job CD
    needs: ci                           # solo se ejecuta si ci ha ido bien
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3    # Docker registry login
        with:
          username: ${{ secrets.DOCKERHUB_USER }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build y push
        run: |
          docker build -t miusuario/miapp:latest .
          docker push miusuario/miapp:latest
```

Los **secrets** se crean en *Settings → Secrets and variables → Actions* y se leen con `${{ secrets.NOMBRE }}`. Nunca escribas credenciales dentro del `.yml`.

### 10.8 Publicar en GitHub Pages con Actions (extra)

```yaml
# .github/workflows/pages.yml
name: Deploy Pages
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
    steps:
      - uses: actions/checkout@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .                       # carpeta que se publica
      - uses: actions/deploy-pages@v4
```

Requiere *Settings → Pages → Source: GitHub Actions*.

### 10.9 Runner self-hosted, `run` multilínea y variantes de `uses`

```yaml
# .github/workflows/self-hosted.yml
name: self-hosted
on: [push]
jobs:
  build:
    runs-on: self-hosted                # tu propia máquina
    steps:
      - name: Check out repository      # necesario para usar una action propia del repo
        uses: actions/checkout@v4
      - name: Install and build         # varios comandos en la misma shell
        run: |
          npm ci
          npm run build
      - name: Clean temp directory
        run: rm -rf *
        working-directory: ./temp       # ejecuta el comando en esa carpeta
      - name: Action de terceros (rama por defecto)
        uses: actions/heroku@1.0.0
      - name: Action de terceros (versión concreta)
        uses: actions/aws@v2.0.1
      - name: Action propia del repositorio
        uses: ./.github/actions/my-action
```

Actions de terceros: https://github.com/marketplace?type=actions

## 11. Custom GitHub Actions

Son tareas individuales que se combinan para crear jobs y personalizar el workflow. Se ejecutan dentro de un **step** de un **job** de un **workflow**.

| Tipo | Ventajas | Inconvenientes |
| --- | --- | --- |
| **Docker container** | Más consistente y confiable; no te preocupas por dependencias o herramientas | Solo en runners **Linux**; más lentas |
| **JavaScript** | Código más simple; ejecución más rápida; corre en todos los runners | Debe ser JavaScript puro |
| **Composite** (pasos compuestos) | Agrupa varios steps en una sola action | Cada `run` debe indicar `shell:` |

Las actions necesitan un archivo de **metadatos** `action.yml` (o `.yaml`) que define entradas, salidas y punto de entrada. Convención: `.github/actions/<nombre>/action.yml`.

| Clave | Requerido | Descripción |
| --- | --- | --- |
| `name` | **Sí** | Nombre que muestra GitHub en la pestaña Actions |
| `author` | No | Autor de la action |
| `description` | **Sí** | Descripción breve |
| `inputs` | No | Parámetros de entrada. Atributos: `description`, `required` (requeridos) y `default` (opcional) |
| `outputs` | No | Datos que la action establece; todos sus atributos son requeridos |
| `runs` | **Sí** | Cambia según el tipo (JavaScript, Docker o composite) |
| `branding` | No | `icon` y `color` para el Marketplace (iconos: https://feathericons.com) |

- Por cada input, GitHub crea la variable de entorno `INPUT_<NOMBRE>`, en mayúsculas y con los espacios sustituidos por `_`.
- **`runs` JavaScript:** `using` (requerido, p. ej. `node20`; en clase `node12`), `main` (requerido, fichero principal), `pre` (script anterior a main, condicional con `pre-if`), `post` (posterior a main, condicional con `post-if`).
- **`runs` Docker:** `using: 'docker'` e `image` (requeridos; un `Dockerfile` local o una imagen pública de Docker Hub), `args` (entradas del contenedor), `entrypoint` (invalida el ENTRYPOINT del Dockerfile), `post-entrypoint`.

### 11.1 Ejemplo JavaScript

**Pasos:** crear `action.yml` → crear proyecto Node → instalar librerías → `index.js` → compilar → subir y probar en un workflow.

```yaml
# .github/actions/hello-world-javascript-action/action.yml
name: 'Hello World'
description: 'Greet someone and record the time'
inputs:
  who-to-greet:                    # id del input
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time:                            # id del output
    description: 'The time we greeted you'
runs:
  using: 'node20'
  main: 'dist/index.js'            # fichero compilado (ver más abajo)
branding:
  icon: 'award'
  color: 'green'
```

```bash
cd .github/actions/hello-world-javascript-action
npm init -y                                 # crea el proyecto Node
npm install @actions/core @actions/github   # librerías para inputs/outputs y contexto
# ... crear index.js (siguiente bloque) ...
npm i -g @vercel/ncc                        # compilador para no subir node_modules
ncc build index.js --license licenses.txt   # genera dist/index.js
```

```javascript
// .github/actions/hello-world-javascript-action/index.js
const core = require('@actions/core');
const github = require('@actions/github');

try {
  // input definido en el archivo de metadatos
  const nameToGreet = core.getInput('who-to-greet');
  console.log(`Hello ${nameToGreet}!`);
  const time = (new Date()).toTimeString();
  core.setOutput('time', time);                 // output de la action
  // payload JSON del evento que lanzó el workflow
  const payload = JSON.stringify(github.context.payload, undefined, 2);
  console.log(`The event payload: ${payload}`);
} catch (error) {
  core.setFailed(error.message);                // marca el step como fallido
}
```

Después de compilar, `main` debe apuntar a `dist/index.js`, y la carpeta `dist` se sube al repositorio (`node_modules` no hace falta).

```yaml
# .github/workflows/test-action.yml
on: [push]
jobs:
  hello_world_job:
    runs-on: ubuntu-latest
    name: A job to say hello
    steps:
      - name: Checkout                  # obligatorio para usar una action propia del repo
        uses: actions/checkout@v4
      - name: Hello world action step
        uses: ./.github/actions/hello-world-javascript-action   # en clase: ./ (action en la raíz)
        id: hello                       # id para leer sus outputs
        with:
          who-to-greet: 'Mona the Octocat'
      - name: Get the output time       # usa el output del step hello
        run: echo 'The time was ${{ steps.hello.outputs.time }}'
```

### 11.2 Ejemplo Docker

**Pasos:** crear `Dockerfile` (y el script) → crear `action.yml` → subir y probar en un workflow.

```dockerfile
# .github/actions/hello-docker/Dockerfile
# Imagen del contenedor que ejecuta tu código
FROM alpine:3.10
# Copias tu archivo de código del repositorio
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
# Archivo del código a ejecutar cuando comienza
ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/sh -l
# .github/actions/hello-docker/entrypoint.sh
echo Hello $1                         # $1 = primer arg (inputs.who-to-greet)
time=$(date)
echo time=$time >> $GITHUB_OUTPUT     # output (la forma de clase ::set-output name=time::$time está obsoleta)
```

```yaml
# .github/actions/hello-docker/action.yml
name: 'Hello World'
description: 'Greet someone and record the time'
inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time:
    description: 'The time we greeted you'
runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.who-to-greet }}
```

Se prueba con el mismo workflow del 11.1, cambiando `uses` a `./.github/actions/hello-docker` (el runner debe ser Linux).

### 11.3 Ejemplo composite (extra)

```yaml
# .github/actions/saludo-compuesto/action.yml
name: 'Saludo compuesto'
description: 'Ejecuta varios pasos shell'
inputs:
  nombre:
    description: 'A quién saludar'
    required: false
    default: 'Mundo'
runs:
  using: 'composite'
  steps:
    - run: echo Hola ${{ inputs.nombre }}
      shell: bash                      # obligatorio en cada run de una composite
```

## 12. Repaso final y errores típicos

| Si te piden... | Usa... |
| --- | --- |
| Deshacer sin tocar el historial publicado | `git revert <hash>` |
| Deshacer el último commit conservando cambios | `git reset --soft HEAD~1` |
| Dejar de rastrear un fichero sin borrarlo | `git rm --cached fichero` |
| Traer cambios sin fusionar / fusionando | `git fetch` / `git pull` |
| Crear rama y cambiar a ella | `git checkout -b rama` |
| Subir etiquetas | `git push origin --tags` |
| Ejecutar en varios SO o versiones | `strategy.matrix` + `runs-on: ${{ matrix.os }}` |
| Ejecutar un job después de otro | `needs: job-anterior` |
| Usar una action / un comando | `uses:` / `run:` |

**Errores típicos**

- El workflow no se lanza: ruta distinta de `.github/workflows/`, extensión incorrecta, tabuladores en vez de espacios o evento mal escrito en `on`.
- Olvidar `actions/checkout`: el runner no tiene tu código (tampoco tus actions propias).
- Olvidar que los jobs son **paralelos** por defecto y que cada job usa un runner distinto.
- `git branch` crea la rama pero **no cambia** a ella; `git push` **no** sube las etiquetas; `-u` solo hace falta la primera vez.
- `git reset --hard` **pierde** los cambios; para commits ya publicados usa `revert`.
- `git add .` puede colar ficheros basura: revisa `git status` y usa `.gitignore`.
- Hacer `pull` con cambios sin confirmar: haz commit antes.
- `name` del workflow es opcional; `on`, `jobs`, `runs-on` y `steps` no.
