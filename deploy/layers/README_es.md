# Capas de organización

<!-- hy-mt2-i18n:start -->
[English](./README.md) | [日本語](./README_ja.md) | **Español** | [中文](./README_zh-CN.md) | [한국어](./README_ko.md)
<!-- hy-mt2-i18n:end -->


Este directorio alberga el material de despliegue propio de una organización cuando **qm** se personaliza a partir de un *fork* privado: se trata de un repositorio privado independiente cuya historia comienza como un clon de **qm**, en el cual la versión base permanece idéntica a la del código fuente, y todo lo específico de la organización queda confinado aquí, bajo `deploy/layers/<org>/`.

En el **qm** del código fuente, este directorio solo contiene este archivo, y así sigue siendo. Una capa pertenece únicamente al *fork* privado de una organización y nunca llega al código fuente. La habilidad `upstream-pr` garantiza ese límite; la habilidad `update-qm` fusiona los cambios del código fuente alrededor de ella.

## Crear una capa

```bash
node cli/bin/qm.ts init deploy/layers/<org> --org <slug> --target <fly-or-aws>
```

`qm init` escribe la configuración de despliegue, el ejemplo de nombre de secreto, la estructura básica para el entorno de pruebas y el proveedor, un manual de operaciones, y un archivo `.gitignore` por directorio que impide que los valores de `.env` y el estado de Terraform entren en Git. Genere la capa en lugar de construirla manualmente para que el archivo `.gitignore` venga incluido con ella; el archivo `.gitignore` raíz cubre los mismos archivos como medida de respaldo.

El resultado, descrito en detalle en [`docs/deploy-directory.md`](../../docs/deploy-directory.md), es el siguiente:

```text
deploy/layers/<org>/
  qm.config.jsonc          La configuración de despliegue; se guarda en Git, sin valores de secreto
 .gitignore               Estructura básica; evita que.env y tfstate entren en Git
 .env.example             Nombres de secretos calculados, nunca sus valores
 .env                     Valores locales de secretos; nunca se guardan en Git
  sandbox/                 Herramientas y habilidades de la organización para las computadoras agente
  plugins/<name>/          Imágenes de servicio específicas de la organización
  infra/                   Infraestructura del proveedor y tfvars, en destinos AWS
  slack-app-manifest.yml   Manifiesto del bot generado
  deployment.md            Manual de operaciones
```

Dirija la CLI hacia una capa utilizando `--config`:

```bash
node cli/bin/qm.ts check --config deploy/layers/<org>/qm.config.jsonc
```

Ejecute la CLI desde la estructura del proyecto como se muestra. `npm exec qm` no funciona al hacer un *checkout* de fuentes, ya que el enlace simbólico del espacio de trabajo apunta a `cli/`, que aún no está compilado.

## Directorios cercanos

`deploy/stacks/` contiene configuraciones neutrales para cuentas que se utilizan para probar el backend de Fly, mientras que `deploy/<service>/` alberga la imagen del servicio y las plantillas de Fly que la CLI genera a partir de ellas. Ninguno de estos directorios sirve para almacenar material de organización.

## La regla

Nada dentro de `deploy/layers/` puede llegar al **qm** del código fuente: ni la configuración, ni las herramientas del entorno de pruebas, ni las coordenadas de la infraestructura, ni los nombres de sistemas o personas que aparezcan en ellos. Los secretos nunca entran en Git, ni en este directorio ni en ningún otro. Pertenecen al almacén de secretos cifrados del proveedor, con valores locales únicamente en el archivo `.env` que ha sido marcado como ignorado por Git.
