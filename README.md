# Workflow

Workflows reusables de GitHub Actions, compartidos por todos los servicios de **Snippet Searcher**.

No tiene código ni build. Son archivos YAML que otros repos llaman.

## Qué resuelve

Todos los servicios se verifican igual antes de integrar un cambio. Sin un lugar común, cada repo tendría su propia copia de los mismos pasos, y un arreglo habría que repetirlo en todos.

Acá los pasos se definen **una vez**, y cada servicio los llama en una línea.

## Workflows

| Workflow | Qué hace | Cuándo lo llama un servicio |
|---|---|---|
| `gradle-check.yml` | Baja el código del servicio, instala Java y Gradle, y corre `./gradlew check` | En cada pull request |
| `docker-publish.yml` | Construye la imagen Docker del servicio y la sube a ghcr.io, con el hash del commit como tag | En cada push a `dev` o `main` |

`./gradlew check` es lo que define cada servicio en su build: compilar, tests, ktlint, detekt y cobertura. Este workflow no sabe qué se verifica: solo lo ejecuta.

### `gradle-check.yml`

| Input | Default | Qué es |
|---|---|---|
| `java-version` | `"21"` | Versión de Java con la que se compila |

Corre con `--continue`: si fallan varias verificaciones, el log las muestra todas en una sola corrida, no solo la primera.

Le pasa a Gradle las credenciales para bajar paquetes de GitHub Packages (`GITHUB_ACTOR` y `GITHUB_TOKEN`), con el token que GitHub crea para cada corrida. No hay que configurar ningún secreto.

### `docker-publish.yml`

No tiene inputs. Construye la imagen con el `Dockerfile` que esté en la raíz del servicio y la publica en `ghcr.io/<organización>/<servicio>:<hash del commit>`, todo en minúsculas porque ghcr.io no acepta mayúsculas.

Devuelve el output `image` con el nombre completo de la imagen publicada, para que un paso posterior sepa qué desplegar.

- **El tag es el hash del commit**, nunca `latest`: cada imagen apunta a un commit exacto, y volver atrás es elegir el hash anterior.
- **El token de GitHub Packages entra como secret mount** del build, así que no queda en las capas de la imagen.
- **Permisos:** el workflow que lo llama tiene que conceder `packages: write`. Un workflow reusable no puede tener más permisos que los que le da quien lo llama.

```yaml
name: Publish

on:
  push:
    branches: [dev, main]

jobs:
  publish:
    permissions:
      contents: read
      packages: write
    uses: PPC-INGSIS/workflow/.github/workflows/docker-publish.yml@v3
```

## Cómo usarlo en un servicio

En el repo del servicio, `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
    branches: [dev, main]

jobs:
  check:
    uses: PPC-INGSIS/workflow/.github/workflows/gradle-check.yml@v2
```

Para compilar con otra versión de Java:

```yaml
jobs:
  check:
    uses: PPC-INGSIS/workflow/.github/workflows/gradle-check.yml@v2
    with:
      java-version: "17"
```

## Cómo funciona

- **`on: workflow_call`** hace que un workflow sea reusable: no corre con un push ni con un pull request a este repo, sino cuando otro workflow lo llama.
- **El `checkout` baja el código del repo que llama**, no el de este. Si lo llama `snippet-service`, se compila y se testea `snippet-service`.
- **Los secretos no se heredan solos.** Un workflow reusable no ve los secretos del repo que lo llama, salvo que se los pasen de forma explícita.
- **El archivo tiene que estar en `.github/workflows/`**, en plural. En otra carpeta, GitHub no lo encuentra y no avisa.

## Versiones

Los servicios llaman a una **versión fija** (`@v2`), no a `@main`. Así, un cambio en este repo no rompe a todos los servicios a la vez: cada uno pasa a la versión nueva cuando quiere.

| Referencia | Apunta a | Cuándo usarla |
|---|---|---|
| `@v2` | Un commit fijo | Siempre, en los servicios |
| `@main` | El último commit | Solo para probar un cambio antes de taggearlo |

**Publicar una versión nueva:**

1. Probar el cambio llamándolo con `@main` desde un pull request de un servicio.
2. Si funciona, crear el tag: `git tag v3 && git push origin v3`.
3. Actualizar la referencia en cada servicio.

Un cambio compatible, como agregar un input opcional, no rompe a quienes ya lo usan. Un cambio incompatible, como renombrar un input o hacerlo obligatorio, va con un número de versión nuevo.

## Qué va en este repo y qué no

Un workflow va acá solo si lo usan **dos o más repos**.

| Va | No va |
|---|---|
| Verificar un servicio en cada pull request | El workflow que publica las convenciones de Gradle: lo usa un solo repo, y vive ahí |
| Construir y publicar la imagen Docker de un servicio | Reglas de formato o de cobertura: ya están dentro de `./gradlew check` |
| Desplegar un servicio (pendiente) | |

## Qué falta

- `deploy.yml`: desplegar la imagen publicada.

Los tres forman una cadena: **verificar → empaquetar → desplegar**.
