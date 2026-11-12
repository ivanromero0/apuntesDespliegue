# Ejemplos de GitHub Actions para el examen

Despliegue de aplicaciones web · 2 DAW

## 0. ¿Dónde va ubicado el .yml?

Esto es lo primero que hay que tener claro: **cada tipo de fichero va en una carpeta distinta**.

| Qué es | Ruta dentro del repositorio |
| --- | --- |
| **Workflow** (el que se ejecuta con un evento) | `.github/workflows/nombre.yml` |
| **Custom action** (metadatos de tu propia action) | `.github/actions/nombre-action/action.yml` |
| Código de una custom action (JavaScript) | `.github/actions/nombre-action/index.js` (compilado en `dist/index.js`) |
| Código de una custom action (Docker) | `.github/actions/nombre-action/Dockerfile` y `entrypoint.sh` |

Reglas importantes:

- Los workflows usan sintaxis **YAML** y se guardan en `.github/workflows` (con `.yml` o `.yaml`). Se pueden tener varios workflows, uno por fichero.
- El archivo de una custom action se llama siempre **`action.yml`** (o `action.yaml`).
- Para usar una action que está en tu propio repositorio: `uses: ./.github/actions/nombre-action` (después de `actions/checkout`).
- Los workflows se ejecutan al hacer `git push` de ese fichero, así que hay que hacer `git add`, `git commit` y `git push` para que aparezcan en la pestaña **Actions** del repositorio.

Estructura de ejemplo:

```text
mi-repositorio/
├── .github/
│   ├── workflows/
│   │   ├── tests.yml            <- workflows
│   │   └── deploy.yml
│   └── actions/
│       └── hello-world-javascript-action/
│           ├── action.yml       <- custom action
│           ├── index.js
│           └── dist/index.js
├── src/
└── README.md
```

## 1. Componentes (resumen para recordar)

- **Workflow**: procedimiento automatizado, formado por uno o más jobs, activado por un evento (push, PR...).
- **Job**: conjunto de steps que se ejecutan en el mismo runner. Por defecto los jobs se ejecutan **en paralelo**.
- **Step**: tarea individual dentro de un job. Puede ser una action (`uses`) o un comando de shell (`run`). Se ejecutan **en secuencia**.
- **Action**: comando independiente reutilizable que se combina en steps.
- **Runner**: servidor que ejecuta los jobs. *GitHub-hosted* (Ubuntu, Windows, macOS; gratis 2000 min/mes) o *self-hosted* (tu propia máquina; sin coste pero con mantenimiento).

## 2. Workflow mínimo: name, on, jobs, runs-on, steps

`.github/workflows/primer-ejemplo.yml`

```yaml
name: primer-ejemplo          # (opcional) nombre que sale en la pestaña Actions
on: [push]                    # (obligatorio) evento que lo dispara

jobs:
  check-bats-version:         # identificador del job
    runs-on: ubuntu-latest    # (obligatorio) máquina donde se ejecuta
    steps:
      - uses: actions/checkout@v6        # uses = usar una action
      - uses: actions/setup-node@v6
      - run: npm install -g bats         # run = ejecutar un comando
      - run: bats -v
```

Qué hay que saber explicar:

- `name`: opcional. Si no se pone, GitHub usa la ruta del fichero.
- `on`: requerido. Evento que activa el workflow.
- `jobs`: agrupa los jobs. Cada job necesita un identificador.
- `runs-on`: requerido. Tipo de máquina (github-hosted o self-hosted).
- `uses` = ejecuta una action. `run` = ejecuta un comando.

## 3. Eventos (`on`)

### 3.1 Un solo evento, varios eventos, o con configuración

```yaml
# Un solo evento
on: push
```

```yaml
# Varios eventos (lista)
on: [push, pull_request]
```

```yaml
# Con configuración: solo en la rama main y en tags que empiecen por v
on:
  push:
    branches:
      - main
    tags:
      - 'v*'
  pull_request:
    branches:
      - main
```

### 3.2 Ejecución manual y programada

```yaml
on:
  workflow_dispatch:         # botón "Run workflow" en la pestaña Actions
  schedule:
    - cron: '0 8 * * 1'      # todos los lunes a las 8:00 UTC
```

### 3.3 Filtrar por rutas

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'src/**'             # solo si cambia algo dentro de src/
```

## 4. Ejemplo completo de CI con Node.js

`.github/workflows/ci-node.yml`

```yaml
name: CI Node
on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Descargar el código
        uses: actions/checkout@v6

      - name: Preparar Node.js
        uses: actions/setup-node@v6
        with:
          node-version: '20'          # with = parámetros de la action

      - name: Instalar dependencias
        run: npm ci

      - name: Ejecutar tests
        run: npm test
```

## 5. Varios jobs en paralelo (uno por sistema operativo)

Es el estilo del ejemplo que tienes de Python: tres jobs independientes que corren a la vez.

`.github/workflows/tests.yml`

```yaml
name: Tests
on:
  push:
    branches:
      - main

jobs:
  test-win:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-python@v6
        with:
          python-version: '3.12'
      - run: python -m unittest -v

  test-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-python@v6
        with:
          python-version: '3.13'
      - run: python -m unittest -v

  test-mac:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-python@v6
        with:
          python-version: '3.13'
      - run: python -m unittest -v
```

## 6. Jobs dependientes: `needs`

Por defecto los jobs van en paralelo. Con `needs` se fuerza un orden. Si un job falla, **los que dependen de él no se ejecutan**.

`.github/workflows/needs.yml`

```yaml
name: Pipeline con orden
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: echo "Ejecutando tests..."

  build:
    needs: test               # espera a que termine test (y que sea correcto)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: echo "Compilando..."

  deploy:
    needs: [test, build]      # espera a los dos
    runs-on: ubuntu-latest
    steps:
      - run: echo "Desplegando..."
```

Variante con `always()`: el job se ejecuta **siempre** después de los anteriores, aunque hayan fallado.

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    steps:
      - run: echo "job1"
  job2:
    needs: job1
    runs-on: ubuntu-latest
    steps:
      - run: echo "job2"
  job3:
    if: always()              # se ejecuta pase lo que pase
    needs: [job1, job2]
    runs-on: ubuntu-latest
    steps:
      - run: echo "job3 se ejecuta aunque job1 o job2 fallen"
```

## 7. Variables de entorno: `env`

Se pueden definir a tres niveles: **workflow**, **job** y **step**. Si se repite la misma variable, gana la **más específica** (la del step).

`.github/workflows/env.yml`

```yaml
name: Variables de entorno
on: [push]

env:                           # nivel workflow: la ven todos los jobs
  SERVER: production

jobs:
  saludo:
    runs-on: ubuntu-latest
    env:                       # nivel job: la ven todos los steps de este job
      FIRST_NAME: Mona
    steps:
      - name: Mostrar saludo
        env:                   # nivel step: solo este step
          MY_VAR: Hi there! My name is
          LAST_NAME: Octocat
        run: |
          echo $MY_VAR $FIRST_NAME $LAST_NAME
          echo "Servidor: $SERVER"
```

## 8. Steps con `run`: varios comandos y directorio de trabajo

```yaml
name: Ejemplos de run
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      # Un solo comando
      - name: Instalar dependencias
        run: npm install

      # Varios comandos en el mismo shell (con |)
      - name: Instalación limpia y build
        run: |
          npm ci
          npm run build

      # Ejecutar en otra carpeta
      - name: Limpiar carpeta temporal
        run: rm -rf *
        working-directory: ./temp
```

## 9. Usar actions: `uses` (de terceros y propia)

```yaml
name: Tipos de uses
on: [push]

jobs:
  ejemplo:
    runs-on: ubuntu-latest
    steps:
      # Action oficial/de terceros, con versión concreta (recomendado)
      - name: Descargar código
        uses: actions/checkout@v6

      # Action de terceros del Marketplace con parámetros
      - name: Preparar Java
        uses: actions/setup-java@v5
        with:
          distribution: 'temurin'
          java-version: '21'

      # Action propia, guardada en el mismo repositorio
      # (siempre DESPUÉS de actions/checkout)
      - name: Usar mi action local
        uses: ./.github/actions/my-action
```

Recordatorio: las actions de terceros se buscan en https://github.com/marketplace?type=actions. Se escribe como `propietario/repositorio@versión`.

## 10. Matriz de ejecución: `strategy.matrix`

Crea **varios jobs** a partir de una sola definición, sustituyendo variables.

### 10.1 Matriz simple (versiones de Node)

```yaml
name: Matriz simple
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20, 22]            # se crean 3 jobs
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

### 10.2 Matriz con sistema operativo + versión

Se combinan todas las opciones: 2 sistemas × 3 versiones = **6 jobs**.

```yaml
name: Matriz os x node
on: [push]

jobs:
  test:
    runs-on: ${{ matrix.os }}         # el runner también sale de la matriz
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        node: [18, 20, 22]
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

### 10.3 Matriz con `include` (combinaciones concretas con variables extra)

```yaml
name: Node.js CI
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - node-version: 18.x
            site: "prod"
            datacenter: "site-a"
          - node-version: 20.x
            site: "dev"
            datacenter: "site-b"
    steps:
      - name: Mostrar datos del sitio
        env:
          SITE: ${{ matrix.site }}
          DATACENTER: ${{ matrix.datacenter }}
        run: echo $SITE $DATACENTER
```

Aquí cada ejecución usa su propia versión de Node y sus propias variables de entorno asociadas.

## 11. Runner self-hosted

Solo cambia `runs-on`. El runner es una máquina nuestra que tiene instalada la aplicación de GitHub Actions.

```yaml
name: Runner propio
on: [push]

jobs:
  build:
    runs-on: self-hosted          # nuestra máquina, no la de GitHub
    steps:
      - uses: actions/checkout@v6
      - run: echo "Ejecutando en mi propio servidor"
```

Comparación rápida:

|  | GitHub-hosted | Self-hosted |
| --- | --- | --- |
| Mantenimiento | Ninguno | Lo configuramos y mantenemos nosotros |
| Coste | Se paga por uso (2000 min/mes gratis) | Sin coste por uso |
| Sistemas | Windows, macOS, Ubuntu | El que instalemos |
| Acceso a red local | No | Sí |
| Flexibilidad | Limitada | Total |

## 12. CI + CD: pipeline completo con Docker (secretos)

Ejemplo con dos jobs encadenados: primero CI (tests) y después CD (subir imagen a Docker Hub). Las credenciales **nunca** se escriben en el YAML: se guardan en *Settings → Secrets and variables → Actions* y se leen con `${{ secrets.NOMBRE }}`.

`.github/workflows/ci-cd.yml`

```yaml
name: CI-CD
on:
  push:
    branches:
      - main

jobs:
  ci:                                   # Integración continua
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6       # 1. Descargar código
      - uses: actions/setup-node@v6     # 2. Preparar node
        with:
          node-version: '20'
      - run: npm ci                     # 3. Instalar dependencias
      - run: npm test                   # 4. Tests

  cd:                                   # Despliegue continuo
    needs: ci                           # solo si el CI ha ido bien
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Login en Docker Hub       # 1. Docker registry login
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Construir y subir imagen  # 2. Docker registry push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: miusuario/miapp:latest
```

Relación con el tema: **CI** = integrar y probar con frecuencia; **Continuous Delivery** = el código siempre queda listo para desplegar (pero se despliega a mano); **Continuous Deployment** = cada cambio verificado se despliega solo a producción.

## 13. Desplegar en GitHub Pages con una Action

Combina el tema de Pages con el de Actions: cada push a `main` publica la web.

`.github/workflows/pages.yml`

```yaml
name: Desplegar en GitHub Pages
on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v4
        with:
          path: '.'                     # carpeta con el html
      - id: deployment
        uses: actions/deploy-pages@v4
```

Hay que activar antes *Settings → Pages → Source: GitHub Actions*.

## 14. Custom action en JavaScript

Pasos: (1) crear `action.yml`, (2) crear proyecto Node, (3) instalar librerías, (4) crear `index.js`, (5) compilar, (6) probarla en un workflow.

### 14.1 `action.yml`

Ruta: `.github/actions/hello-world-javascript-action/action.yml`

```yaml
name: 'Hello World'                      # requerido: nombre en la pestaña Actions
description: 'Saluda y devuelve la hora' # requerido
author: 'Iván'                           # opcional

inputs:                                  # opcional: parámetros de entrada
  who-to-greet:
    description: 'Persona a saludar'
    required: true
    default: 'World'                     # default es el único campo opcional

outputs:                                 # opcional: datos que devuelve la action
  time:
    description: 'Hora del saludo'

runs:                                    # requerido
  using: 'node24'                        # runtime de node (en apuntes antiguos: node12/node16/node20)
  main: 'dist/index.js'                  # fichero principal (tras compilar)

branding:                                # opcional: icono y color en el Marketplace
  icon: 'award'
  color: 'green'
```

Cada input se convierte en una variable de entorno `INPUT_<NOMBRE>` en mayúsculas (`INPUT_WHO-TO-GREET`).

### 14.2 Proyecto Node y dependencias

Desde la carpeta `.github/actions/hello-world-javascript-action`:

```bash
npm init -y
npm install @actions/core
npm install @actions/github
```

### 14.3 `index.js` (lógica principal)

```javascript
const core = require('@actions/core');
const github = require('@actions/github');

try {
  // Leer el input definido en action.yml
  const nameToGreet = core.getInput('who-to-greet');
  console.log(`Hello ${nameToGreet}!`);

  // Establecer el output
  const time = (new Date()).toTimeString();
  core.setOutput('time', time);

  // Datos del evento que disparó el workflow
  const payload = JSON.stringify(github.context.payload, undefined, 2);
  console.log(`The event payload: ${payload}`);
} catch (error) {
  core.setFailed(error.message);   // marca el step como fallido
}
```

### 14.4 Compilar (para no subir `node_modules`)

```bash
npm i -g @vercel/ncc
ncc build index.js --license licenses.txt
```

Esto genera `dist/index.js`. Después, en `action.yml` se pone `main: 'dist/index.js'`.

### 14.5 Probarla en un workflow

`.github/workflows/probar-action.yml`

```yaml
name: Probar mi action JS
on: [push]

jobs:
  hello_world_job:
    runs-on: ubuntu-latest
    name: Job que saluda
    steps:
      - uses: actions/checkout@v6                        # imprescindible para ver la action local
      - name: Hello world action step
        id: hello                                        # id para leer sus outputs
        uses: ./.github/actions/hello-world-javascript-action
        with:
          who-to-greet: 'Iván'
      - name: Mostrar el output
        run: echo "La hora fue ${{ steps.hello.outputs.time }}"
```

## 15. Custom action en Docker

Pasos: (1) crear `Dockerfile`, (2) crear `action.yml`, (3) subir los cambios y probarla en un workflow. Solo se ejecutan en runners **Linux** y son más lentas, pero son más consistentes (no dependes de herramientas instaladas).

### 15.1 `Dockerfile`

Ruta: `.github/actions/hello-docker-action/Dockerfile`

```dockerfile
FROM alpine:3.20
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

`entrypoint.sh` (misma carpeta):

```bash
#!/bin/sh -l
echo "Hello $1"
time=$(date)
echo "time=$time" >> $GITHUB_OUTPUT
```

### 15.2 `action.yml`

Ruta: `.github/actions/hello-docker-action/action.yml`

```yaml
name: 'Hello Docker'
description: 'Saluda desde un contenedor'

inputs:
  who-to-greet:
    description: 'Persona a saludar'
    required: true
    default: 'World'

outputs:
  time:
    description: 'Hora del saludo'

runs:
  using: 'docker'
  image: 'Dockerfile'            # Dockerfile local o imagen pública (docker://...)
  args:                          # argumentos que recibe el contenedor ($1)
    - ${{ inputs.who-to-greet }}
```

### 15.3 Probarla en un workflow

`.github/workflows/probar-docker.yml`

```yaml
name: Probar mi action Docker
on: [push]

jobs:
  hello:
    runs-on: ubuntu-latest           # Docker actions: solo Linux
    steps:
      - uses: actions/checkout@v6
      - name: Ejecutar action Docker
        id: hello
        uses: ./.github/actions/hello-docker-action
        with:
          who-to-greet: 'Iván'
      - run: echo "Hora: ${{ steps.hello.outputs.time }}"
```

## 16. Custom action compuesta (composite)

Agrupa varios steps en una sola action. Aquí cada `run` **necesita `shell`**.

Ruta: `.github/actions/saludo-compuesto/action.yml`

```yaml
name: 'Saludo compuesto'
description: 'Varios steps en una sola action'

inputs:
  who-to-greet:
    description: 'Persona a saludar'
    required: true
    default: 'World'

runs:
  using: 'composite'
  steps:
    - run: echo "Hola ${{ inputs.who-to-greet }}"
      shell: bash
    - run: echo "Fecha: $(date)"
      shell: bash
```

## 17. Chuleta final: tipos de custom action

| Tipo | Ventajas | Desventajas |
| --- | --- | --- |
| **Docker container** | Más consistente y fiable; no te preocupas de dependencias | Solo Linux; más lenta |
| **JavaScript** | Código más simple; ejecución más rápida; corre en todos los runners (JS puro) | Hay que compilar o subir `node_modules` |
| **Composite** | Reutiliza varios steps; sin código | Cada `run` necesita `shell` |

## 18. Errores típicos que quitan puntos

- **Indentación**: YAML usa espacios (nunca tabuladores) y 2 espacios por nivel.
- Cada step de la lista empieza con ` -  `. Dentro de un step, `name`, `uses`, `with`, `run` van alineados.
- `runs-on` y `on` son obligatorios; `name` no.
- `uses` usa una action; `run` ejecuta un comando. No se mezclan en el mismo step.
- Para usar una action local hay que hacer **antes** `actions/checkout`.
- Los jobs son paralelos por defecto: si importa el orden, usa `needs`.
- Los secretos van en `${{ secrets.NOMBRE }}`, nunca escritos en el fichero.
- Las expresiones y variables de contexto se escriben `${{ ... }}`.
- El fichero del workflow tiene que estar en `.github/workflows/` o no se ejecutará.
