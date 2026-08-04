# qm

[English](./README.md) | [日本語](./README_ja.md) | **Español** | [中文](./README_zh-CN.md) | [한국어](./README_ko.md)


Una herramienta de agente multijugador para el trabajo. Disponible en Slack y en la web.

![La interfaz web de QM: dos sesiones simultáneas, una barra lateral con archivos personales, cron jobs, llavero digital, despliegues, memoria y habilidades](./docs/screenshots/web-ui-hero.png)

## ¿Qué es QM?

La mayoría de los agentes están diseñados como asistentes personales. Puedes hacer que uno trabaje para toda una empresa, pero rápidamente se vuelve complejo. QM está pensado para startups. Cada empleado cuenta con su propio espacio de trabajo aislado y trabaja de forma independiente sin afectarse mutuamente; además, pueden colaborar con el agente en canales, mensajes grupales y proyectos.

Cada persona y cada sala cuentan con su propia memoria, archivos, vista del llavero digital, permisos, cron jobs, aplicaciones web y entorno aislado específicos.

Está construido teniendo en mente el uso de código abierto. Elige tu propia herramienta y modelo, y cámbialos según sea necesario: Pi, OpenCode, Codex y Claude Code utilizan el mismo núcleo, por lo que un despliegue no está vinculado a ningún proveedor en particular.

## Características

- **Ámbitos personales y compartidos.** Las personas personalizan el agente para que sea *suyo*, y aun así pueden trabajar en colaboración en canales de Slack y proyectos.
- **Slack y web.** La misma identidad y configuración se mantienen tanto en Slack como en la aplicación web.
- **Control administrativo.** Configura parámetros a nivel de organización, la postura de seguridad y qué herramientas y modelos están disponibles.
- **Aplicaciones web.** Crea aplicaciones internas personalizadas y distribúyelas entre las personas adecuadas.
- **Habilidades compartidas.** Las habilidades pertenecen a un ámbito específico y pueden compartirse mediante autorización; existe también la posibilidad de promoverlas a toda la organización bajo supervisión administrativa, además de importar paquetes de habilidades desde repositorios Git.
- **Trabajo en segundo plano.** Los cron jobs y monitores ejecutan tareas mientras nadie está mirando.

## Qué puedes hacer con él

- Buscar notas internas, correos electrónicos, documentos, bases de datos y contenido de la web de forma conjunta.
- Obtener información del “cerebro” de tu empresa.
- Crear aplicaciones internas, distribuirlas entre las personas adecuadas y mantener sus datos actualizados.
- Aprender tu estilo de escritura a partir de mensajes anteriores y luego organizar tu bandeja de entrada según un horario establecido, incluyendo etiquetas y borradores de respuesta.
- Trabajar en un repositorio existente: ejecutar pruebas, abrir solicitudes de integración, monitorear procesos CI y revisar registros del sistema.
- Seguir el progreso de un proyecto en un canal compartido y publicar actualizaciones y seguimientos.

## Arquitectura

```mermaid
flowchart LR
  DB[("Postgres<br/>sesiones · memoria · cola")]

  subgraph CORE["Núcleo sin interfaz gráfica"]
    API["API · identidad · políticas · planificador"]
    LOOP["Bucle del agente<br/>(Pi, OpenCode, Claude Code)"]
    API <--> LOOP
  end

  SBX["Entorno aislado por ámbito<br/>archivos · herramientas · servicios conectados"]

  DB <--> API
  LOOP <--> SBX
```

Cada acción pasa por un núcleo central, que puede utilizar diversos modelos y herramientas para generar la respuesta. Una capa de persistencia basada en Postgres almacena los datos de los usuarios, el historial de sesiones y otros estados duraderos. El agente dispone de una pequeña superficie de herramientas fija; una de ellas es `execute`, que ejecuta comandos en el entorno aislado propio del ámbito —su computadora duradera, donde las herramientas instaladas permanecen siempre allí. La interfaz web, el panel administrativo y el portal público son complementos opcionales sobre la API HTTP del núcleo; Slack es un complemento opcional que se ejecuta dentro del proceso y que el núcleo inicia y supervisa mediante un cliente de servicio directo.

El núcleo ejecuta TypeScript directamente en Node y utiliza Fastify para HTTP. El complemento de Slack emplea Bolt; la interfaz web se construye con Vite y se renderiza con Lit.

El núcleo en sí es genérico. Todo lo específico de una empresa —configuración de la organización, herramientas y habilidades personalizadas, imagen del entorno aislado, infraestructura— se encuentra en un **directorio de despliegue** que la CLI [`qm`](./cli/README.md) valida y despliega. Cada componente (herramienta, almacenamiento de sesiones, entorno aislado, memoria) está detrás de una interfaz, por lo que las implementaciones en producción se intercambian mediante un único archivo de configuración.

## Seguridad y secretos

El enfoque de QM sigue el modelo de agentes de codificación locales como OpenCode, Codex y Claude Code: el agente actúa como la persona para quien trabaja, utilizando sus credenciales y permisos, y todo lo que hace está sometido a auditoría. Una organización elige una postura de seguridad, y los ámbitos más restringidos solo pueden endurecerla aún más:

- **Estricto** —cada llamada a una herramienta se detiene para obtener aprobación humana, excepto los dos mecanismos que terminan la sesión sin efecto alguno.
- **Automático** (por defecto) —un clasificador examina los datos externos etiquetados y los resultados de las herramientas antes de que lleguen al modelo; un despliegue puede dirigir ese proceso a su propio proxy de filtrado.
- **Peligroso** —no hay filtrado de contenido ni pausas entre llamadas a herramientas.

La política de comandos predefinida —reglas de aprobación y denegaciones explícitas para operaciones como eliminaciones recursivas o consultas SQL destructivas— se aplica en todos los modos, incluido el modo “Peligroso”.

[`SECURITY.md`](./SECURITY.md) contiene el modelo de amenazas, las suposiciones sobre el operador y las limitaciones conocidas.

## Despliégalo en tu organización

Crea un repositorio de despliegue propiedad de la organización que dependa de `@yc-software/qm`:

```bash
npm exec --yes --package=@yc-software/qm@latest -- \
  qm init. --org <slug> --target <fly-or-aws>
npm install
```

La inicialización crea una habilidad de despliegue para el agente y aborda aspectos como la infraestructura, el inicio de sesión en la web, las credenciales de los conectores, el acceso opcional a Slack, el despliegue y la verificación en tiempo real; no es necesario descargar el código fuente. Cada despliegue se ejecuta en la cuenta de nube del operador; la inicialización no genera ni activa procesos CI de despliegue, y este repositorio no cuenta con un flujo de trabajo de despliegue en producción. Consulta [`deployment.md`](./deployment.md) para más detalles.

## Contribuciones

Aceptamos contribuciones en forma de texto escrito por personas, no de código; consulta [`CONTRIBUTING.md`](./CONTRIBUTING.md). Describe informalmente el cambio que deseas realizar en un archivo `.txt` o `.md` dentro de [`adrs/`](./adrs/); si estamos de acuerdo, nos encargaremos de su implementación. Informa sobre vulnerabilidades de forma privada; consulta [`SECURITY.md`](./SECURITY.md), no mediante un problema público.

## Personaliza tu instancia

El repositorio de despliegue mencionado anteriormente ya incluye configuración y una capa de entorno aislado, por lo que nunca es necesario descargar el código fuente. Algunas organizaciones prefieren lo contrario: tener todo el código base en un único lugar, de modo que los ingenieros y los agentes de codificación puedan leer tanto el código base como las personalizaciones al mismo tiempo, manteniendo las personalizaciones en privado. Para ello, crea un **fork privado**: un repositorio privado independiente cuya historia comienza como una copia de qm y cuyo núcleo permanece idéntico al original.

Rellénalo una vez y luego clónalo para usarlo:

```bash
gh repo create <org>/qm-private --private

git clone --bare git@github.com:yc-software/qm qm-seed.git
git -C qm-seed.git push --mirror git@github.com:<org>/qm-private
rm -rf qm-seed.git

git clone git@github.com:<org>/qm-private
git -C qm-private remote add upstream git@github.com:yc-software/qm
```

Crea el fork privado mediante una clonación simple, como se muestra arriba, y nunca usando la función de fork de GitHub. Aquí la palabra “fork” se refiere al concepto de una copia derivada que se separa intencionadamente del original y se fusiona con él, no al botón “Fork” de GitHub. Un fork de GitHub hereda la visibilidad del repositorio del que proviene, por lo que un fork de un repositorio público no puede hacerse privado. Además, un fork de GitHub comparte la misma red de objetos que el repositorio original, por lo que los commits enviados al fork siguen siendo accesibles mediante SHA desde el lado público. Muchas organizaciones también prohíben crear forks de repositorios privados. Una clonación simple no presenta ninguno de estos problemas, aunque tiene una desventaja: el clon resultante es un repositorio normal, por lo que los flujos de trabajo CI del origen se ejecutan en tu propia cuenta. Deberás proporcionar los secretos que esos flujos necesitan, o desactivar aquellos que no quieras que se ejecuten.

Todo lo específico de tu organización debe colocarse en `deploy/layers/<org>/` —configuración, herramientas y habilidades del entorno aislado, imágenes de complementos, infraestructura— siguiendo la misma estructura que genera `qm init`. Consulta [`deploy/layers/README.md`](./deploy/layers/README.md). El núcleo permanece idéntico en bytes al original, lo que permite que las fusiones sean pequeñas.

Dos habilidades mantienen la separación en ambas direcciones. `update-qm` fusiona el qm del origen en el fork privado e abre la solicitud de sincronización; `upstream-pr` envía una corrección independiente de la organización de vuelta a qm, cortando la rama desde `upstream/main` y revisando la diferencia generada, los mensajes de commit y las capturas de pantalla en busca de identificadores de la organización antes de hacer el push. Nada que esté bajo `deploy/layers/` llega nunca al origen.

## Profundizando más

- [`docs/getting-started.md`](./docs/getting-started.md) —primera ejecución, de principio a fin.
- [`cli/README.md`](./cli/README.md) —la CLI `qm` y el contrato del directorio de despliegue.
- [`docs/deploy-directory.md`](./docs/deploy-directory.md) —el directorio de despliegue en detalle.
- [`.env.example`](./.env.example) —toda configuración documentada en su lugar correspondiente.
- [`plugins/`](./plugins) —las interfaces (Slack, interfaz web, administración, portal).

## Licencia

Salvo que se indique lo contrario, QM está disponible bajo la [Licencia MIT](./LICENSE).
