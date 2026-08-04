# `qm`

<!-- hy-mt2-i18n:start -->
[English](./README.md) | [日本語](./README_ja.md) | **Español** | [中文](./README_zh-CN.md) | [한국어](./README_ko.md)
<!-- hy-mt2-i18n:end -->


La CLI de despliegue autónoma para QM. El esquema del directorio normativo, las garantías de seguridad, el comportamiento esperado y el ciclo de vida se encuentran en
[`docs/deploy-directory.md`](../docs/deploy-directory.md). La orden `qm init` convierte el manual de uso del paquete consumible por los agentes en el repositorio de despliegue.

```bash
npm exec --yes --package=@yc-software/qm@latest -- \
  qm init. --org acme --target aws
npm install
npm exec qm -- check
npm exec qm -- infra render
npm exec qm -- doctor
npm exec qm -- infra build-image
npm exec qm -- plan
npm exec qm -- up --yes
npm exec qm -- check --live
```

Este paquete se publica en npm bajo el nombre `@yc-software/qm`, y la información de origen proporcionada por npm certifica el flujo de trabajo de compilación. Una nueva versión se genera al ejecutar `.github/workflows/release.yml` desde la rama `main`: este proceso firma e inserta las imágenes generadas internamente, publica los valores fijos del paquete junto con sus hashes, asigna la etiqueta `v<versión>` y crea la versión en GitHub con dichos hashes adjuntos. La versión proviene de `cli/package.json`; el sistema CI requiere una solicitud de pull request cada vez que se modifica lo que incluye el paquete; si ya existe una etiqueta con esa versión, el proceso de lanzamiento se detiene sin intentar actualizarla. El manifiesto de la imagen guardada sirve como indicador, y el despliegue puede sobrescribirlo con los hashes reales. La prueba `packed-artifact` verifica localmente el camino de acceso al paquete por parte del cliente.

Esta CLI se utiliza para desplegar servicios de QM que ejecutan tareas durante mucho tiempo; no es el entorno de ejecución en sí. Docker los ejecuta localmente, Fly los gestiona como aplicaciones Fly utilizando Fly Machines como computadoras para los agentes, y AWS ejecuta tareas ARM64 con hashes fijados en ECS Fargate, empleando computadoras agentes Lambda MicroVM.

## Directorio de despliegue

```text
qm.config.jsonc
package.json
package-lock.json
deployment.md
.codex/skills/deploy-qm/
.env.example
.env
slack-app-manifest.yml
slack-sso-manifest.yml
sandbox/
  tools/<id>/tool.json
  tools/<id>/<binary>
  skills/<id>/SKILL.md
  Dockerfile
plugins/<name>/Dockerfile
infra/
```

El archivo `qm.config.jsonc` se guarda en el repositorio y no contiene valores confidenciales. El archivo `.env` se ignora. El archivo `package.json` fija la versión exacta del paquete de la CLI a la utilizada para crear el directorio; el valor `contract: 1` solo sirve como mínimo de compatibilidad, por lo que cada vez que se descarga el código se utiliza siempre el mismo intérprete; actualice deliberadamente ese valor fijo. Al entrar en esa carpeta, las órdenes DEPLOY actúan sobre ella; las opciones `--config` / `--env-file` / `--sandbox-dir` permiten cambiar la ubicación de ciertos componentes (por ejemplo, varios despliegues que comparten un mismo directorio `sandbox/`). La orden `check` valida la configuración, los nombres de los secretos calculados, las herramientas, las habilidades y los plugins, todo sin necesidad de conexión a red; las órdenes `up`, `plan` y `sandbox build` realizan primero las mismas verificaciones. La orden `doctor` comprueba los requisitos externos de forma solo de lectura. La orden `plan` genera el plan de despliegue; para realizar cambios en AWS se necesita ejecutar `up --yes`.

En AWS, la orden `up` toma una instantánea de la instancia RDS antes de realizar la primera modificación, le da un nombre basado en el manifiesto de despliegue que la precede y la registra en dicho manifiesto. La orden `rollback` solo restaura el código y la configuración, por lo que muestra esa instantánea como el punto de restauración correspondiente (`aws rds restore-db-instance-from-db-snapshot`). Las instantáneas creadas antes del despliegue se limitan a un número determinado; la opción `aws.predeployDbSnapshot: false` desactiva esta restricción.

La orden `sandbox build` realiza una compilación con fines de validación local. La orden `sandbox publish` envía la imagen a través del registro OCI configurado, calcula los hashes de la imagen y de su base, registra el valor fijo de la base en la configuración y el valor fijo de la imagen en la configuración (en formato docker/fly) o en el manifiesto duradero de despliegue de AWS, sincroniza la capa de despliegue duradera cuando es posible acceder al núcleo del sistema y vuelve a dirigir el tráfico hacia el núcleo Fly o AWS en ejecución. En AWS se requiere la opción `sandbox.backend: "sprites"`, y antes de generar cualquier imagen se necesita un manifiesto de despliegue existente; además, no se debe utilizar la opción `sandbox.image` para sobrescribirlo, ya que dicha opción solo se utiliza la primera vez que se ejecuta `qm up` y debe eliminarse posteriormente. Cada ejecución normal de `up` también realiza la sincronización de la capa.

Por defecto, se utiliza el clasificador de modelos integrado, a menos que en `qm.config.jsonc` se especifique un proxy `securityScreen` con una etiqueta de proveedor, un endpoint HTTPS y una opción de implementación `shadow` o `enforce`. El token del proxy se envía por separado a través de la variable `secretEnv.core.SECURITY_SCREEN_PROXY_TOKEN`.

## Órdenes

```text
init [dir] [--org id] [--target docker|fly|aws]
check [--json] [--live]
doctor
infra render|build-image|delete-image|delete-task-definitions
conformance [dir] [--static]
plan
up [--yes] [--build-from[=repo]] [--image-label label]
slack render
outputs [--json]
proof scope-key <scope-id>
secrets push [--from file]
status
logs [service] [-f] [--tail n]
down [--purge]
rollback [--to revision-or-sha]
sandbox build [--from image] [--tag tag] [--dry-run]
sandbox publish [--from image] [--app registry/repo] [--tag tag] [--dry-run]
```

Todas las órdenes de despliegue aceptan las opciones `--config`, `--env-file` y `--sandbox-dir`. La opción `dev` sigue siendo el entorno de trabajo para los colaboradores y está separada del contrato de despliegue portátil.

## Contrato del paquete

La exportación `@yc-software/qm/contract` representa la interfaz programática disponible para las pruebas de conformidad. Expose la versión del contrato, las funciones de análisis y renderizado, así como los identificadores de los proveedores, sin necesidad de registrar plugins de ejecución arbitrarios. Cualquier cambio en el directorio que genere incompatibilidades aumenta el número mayor de la versión del contrato; dentro de cada número mayor se pueden añadir campos opcionales.

El paquete no tiene dependencias de ejecución. Utiliza herramientas como Docker con Buildx, Flyctl, la CLI de AWS y Git. Terraform se ejecuta por parte del operador contra el módulo generado mediante la orden `init`.
