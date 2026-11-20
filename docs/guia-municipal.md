# Guía municipal de instalación y operación del Sistema de Entrega a Recepción (SER)

Esta guía cubre dos opciones:
- **Opción B - Distribución mediante imagen Docker**
- **Opción C - Máquina virtual con SER ya desplegado**

La herramienta de administración (CLI) en adelante "ser-qro-cli" administra el sistema SER, que ya se distribuye como imágenes privadas de Docker.

## Glosario

| Término | Significado en esta guía |
| --- | --- |
| **PAT** | El Personal Access Token individual (PAT) de GitHub, es token emitido para el municipio con permisos de descarga para la imagen. Se captura cuando el CLI o la herramienta de primera instalación solicita credenciales de Docker. |
| **Frase de paso de GPG** | GNU Privacy Guard (GPG) es una frase secreta definida por el usuario que protege la clave privada GPG usada por `pass` (el administrador de contraseñas estándar de Unix). Pinentry (gestor seguro de contraseñas) la solicita de forma interactiva. |
| **GHCR** | GitHub Container Registry, registro privado donde se publican las imágenes aprobadas del sismtea SER y de laherramienta de administración CLI. |
| **Asistente de credenciales Docker** | Programa que entrega credenciales a Docker. En esta guía, `docker-credential-pass` guarda la credencial de GHCR cifrada en el almacén `pass`. |
| **`.env` del sistema SER** | Archivo de configuración de la aplicación y Compose necesario para el funcionamiento del sistema SER y el correcto despliegue del mismo. |
| **Usuarios `ser_bootstrap` / `ser_app`** | Son roles de PostgreSQL: `ser_bootstrap` administra toda la base de datos; `ser_app` es solo de propietario de la base de datos SER que usa Laravel y no tiene privilegios globales de superusuario. |
| **Volumen** | Almacenamiento persistente administrado por Docker para la base de datos, los archivos y el almacenamiento de Laravel. Tener cuidado de no eliminarlo al recrear contenedores si no existe riesgo de perdida de información. |

## Perfiles de recursos SER

La herramienta de administración (CLI) ofrece tres perfiles predefinidos y un perfil personalizado `custom`. Los perfiles fijan presupuestos y límites de CPU/memoria para los contenedores; no detectan la carga de usuarios ni sustituyen pruebas de capacidad. La tabla supone **una réplica de cola** y las reservas predeterminadas de host (`0.5` CPU lógica y `25%` de la RAM):

| Perfil | Presupuesto SER (CPU / RAM) | Límites sumados de contenedores con 1 cola (CPU / RAM) | Mínimo teórico del host* |
| --- | ---: | ---: | ---: |
| `small` | 2 CPU / 3 GiB | 2 CPU / 2.75 GiB | 3 CPU lógicas / 4 GiB RAM |
| `medium` | 3.5 CPU / 6 GiB | 3.5 CPU / 5.75 GiB | 4 CPU lógicas / 8 GiB RAM |
| `large` | 6 CPU / 12 GiB | 6 CPU / 10.5 GiB | 7 CPU lógicas / 16 GiB RAM |
| `custom` | Definido por el municipio | Depende de los límites ingresados y las réplicas | Se calcula a partir de los presupuestos y reservas elegidos |

\* Es el mínimo matemático que pasa la validación del CLI con las reservas indicadas: debe quedar el presupuesto del sistema SER después de reservar CPU y RAM al host. No es una recomendación de dimensionamiento ni certifica que Docker, Compose, el sistema operativo o las imágenes del sistema SER sean compatibles. Para operación real puede requerirse más capacidad según concurrencia, tamaño de datos, tareas y crecimiento.

El perfil aplica límites por servicio. Con una réplica de cola, `small` asigna 0.75 CPU/640 MiB a `app`, 0.50 CPU/768 MiB a `queue`, 0.25 CPU/128 MiB a `nginx` y 0.50 CPU/1280 MiB a PostgreSQL. `medium` asigna 1.25/1536 MiB, 0.75/1 GiB, 0.25/256 MiB y 1.25/3 GiB, respectivamente. `large` asigna 2/2 GiB, 1/2 GiB, 0.5/512 MiB y 2.5/6 GiB. Las asignaciones de `queue` se multiplican por el número de réplicas; al aumentarlas, revisa y posiblemente ajusta los presupuestos mediante `custom`.

La opción C parte del perfil `small`. Para consultar o cambiar el perfil, sigue la sección [Consultar y ajustar recursos](#consultar-y-ajustar-recursos). El comando `resources set custom` solicita límites, presupuestos, réplicas y reservas, y valida la combinación antes de guardar.

## 1. Qué se entrega y dónde vive

En el opción B, se entrega un archivo `.tar.gz` versionado parsa instalación de la herramienta de administración CLI, su checksum y esta guía por el canal institucional aprobado. El paquete contiene un script Bash, esta guía y sumas de comprobación.

El directorio estándar del despliegue es `/opt/web/ser-qro`.

| Ruta | Uso | Permisos recomendados |
| --- | --- | --- |
| `/opt/web/ser-qro/.env` | Configuración municipal del sistema SER; incluye secretos de Laravel y `ser_app`, no el PAT de GHCR. | `root:root`, `0600` |
| `/opt/web/ser-qro/docker-compose.yaml` | Servicios y montajes del sistema SER. | `root:root`, `0644` |
| `/opt/web/ser-qro/docker-compose.pgadmin.yaml` | Override opcional para publicar PostgreSQL solo en loopback. | `root:root`, `0644` |
| `/opt/web/ser-qro/secrets/ser_bootstrap_password` | Clave de PostgreSQL para usuario `ser_bootstrap`; no va en `.env` ni en la imagen. | Archivo `root:root`, `0600`; carpeta `0700` |
| `/opt/web/ser-qro/services/` e `init-scripts/` | Configuración de PHP/PostgreSQL y creación inicial del rol de aplicación. | `root:root`; montajes de solo lectura |
| `/opt/web/ser-qro/cli/runtime/` | Versiones del ejecutable del CLI; `current` selecciona la activa. | `root:root` |
| `/usr/local/bin/ser-qro-cli` | Comando global, enlazado al runtime anterior. | Administrado por el instalador |
| `/opt/web/ser-qro/backups/` | Ubicación predeterminada de respaldos. | `0700`; por defecto comparte el disco de la VM |

Los datos PostgreSQL, archivos subidos y almacenamiento Laravel viven en volúmenes con nombre de Docker (`ser-pgsql`, `ser-files` y `ser-storage`), no en carpetas host vacías. Deja que las imágenes creen sus permisos y propietarios. No borres volúmenes ni agregues montajes por cuenta propia.

`services/postgresql/` debe poder leerse por el usuario no-root del contenedor; el scaffold lo crea con permisos `0755` y usa el montaje de solo lectura `:ro,Z`. `Z` solicita una etiqueta SELinux privada en hosts que aplican SELinux, como RHEL/Rocky. En Ubuntu se mantienen los permisos Unix y el montaje de solo lectura.

El comando global del CLI se llama `ser-qro-cli`. El CLI no mantiene un segundo `.env`: lee el del despliegue para operar SER y conserva sus propios logs/lock en archivos separados.

## 2. Requisitos y compatibilidad

Antes de comenzar, confirma:

- Arquitectura `x86_64`/`amd64` o `aarch64`/`arm64`.
- Docker Engine instalado y activo, más el complemento Docker Compose v2. El asistente municipal **no instala Docker**, no habilita su servicio y no añade usuarios al grupo `docker`.
- Acceso HTTPS saliente a GHCR y a los registros de imágenes que usa la entrega del sistema SER; la plantilla actual obtiene `postgres:18-alpine` de Docker Hub.
- Una cuenta autorizada para administrar Docker y escribir en `/opt/web/ser-qro` y `/usr/local/bin`.
- Una sesión de terminal interactiva durante la configuración de GPG, Pinentry y el PAT.

Instala Docker Engine y Compose desde las fuentes oficiales o aprobadas por el municipio. El asistente informa si faltan, pero no cambia repositorios, firewall ni servicios:

- [Docker Engine en RHEL](https://docs.docker.com/engine/install/rhel/)
- [Docker Engine en Rocky Linux](https://docs.docker.com/engine/install/centos/)
- [Docker Engine en Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

Comprueba el servidor con `uname -m`, `docker version`, `docker compose version` y `docker info`. El script `ser-qro-bootstrap.sh --check` también revisa la arquitectura, detecta `dnf`, `apt-get` o `yum` y comprueba dependencias; no instala ni modifica nada. Reconocer un gestor permite preparar paquetes auxiliares, pero no certifica que la distribución, Docker/Compose o las imágenes del sistema SER sean compatibles. Un resultado distinto de cero indica que falta algún componente o no se encontró un gestor contemplado.

### Matriz de compatibilidad

| Componente o sistema | Estado y evidencia |
| --- | --- |
| Binario del CLI | Se construye para `x86_64` y `aarch64` con baseline `manylinux_2_28` (glibc 2.28). CI prueba el ejecutable dentro de manylinux y en runners de ambas arquitecturas. |
| RHEL 9/10 y Rocky Linux 9/10 | Sistemas recomendados para despliegue; el asistente detecta el gestor disponible, pero eso no certifica el despliegue completo. |
| Ubuntu 22.04/24.04 LTS | Sistemas recomendados para despliegue. La versión LTS vigente debe validarse con la versión de Docker/Compose y las imágenes del sistema SER antes de certificarla. Ubuntu 26.04 y posteriores no están certificados hasta contar con pruebas. |
| Rocky Linux 10.2 | Plataforma prevista para el opción C; la entrega debe validarse en la VM antes de considerarla probada. |
| Otras distribuciones con `dnf`, `apt-get` o `yum` | El asistente puede detectar el gestor e instalar paquetes auxiliares desde repositorios ya configurados. Esto no declara compatible la distribución ni certifica Docker, Compose o SER. |
| Docker Engine, Compose e imágenes del sistema SER | Son criterios distintos y requieren pruebas propias. Que el binario funcione con glibc 2.28 **no** certifica estos componentes ni una distribución. |
| CentOS Linux 8 u otras distribuciones/versiones | No se declaran compatibles sin soporte vigente de Docker/Compose y pruebas del sistema SER. |

## 3. Opción B: Distribución mediante imagen Docker

### 3.1 Revisar y ejecutar el paquete

Confirma que recibiste el `.tar.gz`, su archivo `.sha256` y esta guía por el canal institucional autorizado. Verifica el archivo antes de extraerlo; el checksum debe entregarse por un canal confiable separado del archivo que valida:

```bash
# Nota: reemplaza `X.Y.Z` con la versión correspondiente del paquete descargado.

# Verifica la integridad del archivo descargado comparando su hash SHA256
sha256sum --check ser-qro-bootstrap-cli-vX.Y.Z.tar.gz.sha256

# Extrae el contenido del paquete comprimido
tar -xzf ser-qro-bootstrap-cli-vX.Y.Z.tar.gz

# Ingresa al directorio creado durante la extracción
cd ser-qro-bootstrap-cli-vX.Y.Z

# Verifica la integridad de todos los archivos incluidos en el paquete
sha256sum --check SHA256SUMS

# Ejecuta las validaciones previas del instalador sin realizar cambios en el sistema
bash ser-qro-bootstrap.sh --check
```

`--check` es de solo lectura. Es normal que reporte Cosign, `docker-credential-pass` o el CLI como faltantes antes de preparar el servidor. El asistente necesita detectar un gestor de paquetes implementado y Docker/Compose activo.

El asistente requiere terminal interactiva y privilegios administrativos. Ejecuta el script desde una sesión local o SSH interactiva:

```bash
sudo bash ser-qro-bootstrap.sh
```

Lee cada etapa y confirma solo los cambios autorizados. No lo ejecutes con entrada redirigida, `sudo -n` ni como `curl ... | sh`.

### 3.2 Qué hace el asistente

El asistente muestra dos fases con encabezados visuales: **Etapa 1** instala el CLI y sus herramientas auxiliares; **Etapa 2** prepara la configuración y ofrece desplegar el sistema SER. La etapa 2 solo se marca como finalizada si el despliegue termina correctamente. Si se pospone, queda claramente indicada como pendiente y se puede volver a retomar posteriormente usando el comando `ser-qro-cli deploy`.

Después de confirmar, el script:

1. Instala solo paquetes auxiliares (`curl`, `jq`, `pass`, GPG, OpenSSL y Pinentry) desde los repositorios que ya estén configurados. No habilita CRB/EPEL/Universe ni añade repositorios.
2. Descarga Cosign `3.1.4` y `docker-credential-pass` `0.9.13` para la arquitectura detectada; valida los SHA256 publicados para esos assets antes de instalarlos. Una versión diferente requiere revisión y actualización del paquete del CLI.
3. Configura Pinentry y `pass` para la cuenta `root`, y asocia GHCR con `docker-credential-pass`. Si `pass` ya tiene una huella GPG configurada, la reutiliza; si hace falta crear una clave, Pinentry solicita la **frase de paso de GPG** en la terminal.
4. Reutiliza una credencial GHCR que ya pueda recuperarse desde `pass`; si no existe, solicita el PAT mediante el prompt de `docker login`. Usa el PAT individual que se te asigno; no lo pegues en comandos, URLs, variables de entorno ni historial.
5. Descarga la imagen fijada del CLI, comprueba su firma Cosign y solo entonces la instala. Después copia los archivos faltantes de la plantilla mantenida a `/opt/web/ser-qro`.
6. Solicita una versión del sistema SER aprobada y `APP_URL`; crea un `.env` nuevo desde `.env.example`, genera valores aleatorios para `APP_KEY` y `DB_PASSWORD`, y crea `secrets/ser_bootstrap_password`. No muestra esos secretos.
7. Ofrece configurar UFEA y correo. Puedes omitir cualquiera de las secciones y completarla después en `/opt/web/ser-qro/.env`; para aplicar cambios después del despliegue se usa `ser-qro-cli config apply`, que recrea los servicios afectados y causa una interrupción breve.
8. Ejecuta `ser-qro-cli doctor` y, con una confirmación aparte, puede iniciar el primer despliegue.

Al completar UFEA, se solicita la URL base y el token; el token se captura dos veces sin eco y solo se conserva si ambas entradas coinciden. En correo, se solicitan las variables `MAIL_*` que usa la plantilla: el username y password pueden dejarse vacíos si el servidor no requiere autenticación. La contraseña SMTP también se captura dos veces sin eco. Los campos no secretos se muestran para confirmar antes de escribirlos y sus nombres se destacan en color cuando la terminal lo admite. Durante una sección opcional, **R** reintenta el dato, **C** cancela toda esa sección y continúa sin guardar ninguno de sus valores, y **S** sale explícitamente del asistente. Los valores se guardan en `.env` con quoting compatible con Compose; las preguntas no prueban conectividad con UFEA ni con el servidor SMTP.

El script no instala Laravel en el host ni ejecuta `composer install`; copia la plantilla Compose mantenida por el proyecto. No cambia el firewall ni publica PostgreSQL. No reemplaza un `.env` existente ni la configuración de Docker/GPG incompatible: se detiene y explica el conflicto. Si encuentra una credencial directa de GHCR en Docker, solicita quitar el login guardado con `docker logout ghcr.io` como `root`; puede ser necesario volver a ingresar el PAT, pero esto no cierra sesión del sistema ni afecta contenedores. El log técnico del asistente se crea con permisos root-only bajo `/var/log/ser-qro-cli/`; no contiene el PAT ni valores de `.env`. Un fallo puede dejar instalados paquetes auxiliares o archivos ya confirmados; revisa el mensaje y el log antes de reintentar.

El directorio `/opt/web/ser-qro` debe estar libre de `docker-compose.yaml` y `.env` para el primer bootstrap automatizado. Si ya existe configuración municipal, no ejecutes el asistente de primer despliegue; continúa con los comandos del CLI y la guía de operación. Si el asistente se interrumpe después de copiar la plantilla, no lo ejecutes de nuevo: revisa los archivos existentes y termina manualmente con `ser-qro-cli doctor`.

### 3.3 Revisar configuración y primer despliegue

El asistente crea la configuración solo a partir de la plantilla versionada del proyecto. Revisa los valores municipales solicitados y confirma que la versión del sistema SER esté aprobada y fijada. Los secretos se dejan en `.env` con modo `0600`; la contraseña de `ser_bootstrap` se deja en `secrets/ser_bootstrap_password` con modo `0600`.

`APP_URL` es la URL completa de Laravel, con `http://` o `https://`. El host pueder ser el nombre DNS o una IPv4, por ejemplo `https://ser.municipio.gob.mx` o `http://10.4.10.4`; no ingreses solo el nombre o la IP. Acuerda el esquema, puerto y nombre público con quien administre el proxy y TLS.

El primer despliegue se hace con:

```bash
ser-qro-cli doctor
ser-qro-cli resources show
ser-qro-cli deploy
```

`deploy` verifica las firmas Cosign de las imágenes fijadas, descarga imágenes, inicia los servicios, espera a PostgreSQL y ejecuta `php artisan migrate --force` y el seed inicial únicamente si la base es realmente nueva y la protección persistente lo permite. SER ya viene empaquetado con PHP, Composer y sus dependencias; no ejecutes `composer install` ni `migrate:fresh` en el servidor.

Si `deploy` falla durante `docker compose pull` con un timeout hacia `registry-1.docker.io`, la imagen pública de PostgreSQL (`postgres:18-alpine`) no pudo descargarse desde Docker Hub. En ese punto el CLI aún no inicia servicios ni ejecuta migraciones o seed. Comprueba la conectividad del host hacia Docker Hub y, cuando se restablezca, reintenta `ser-qro-cli deploy`; no hace falta borrar la configuración ni los volúmenes. Si quieres aislar la descarga, ejecuta `sudo docker pull postgres:18-alpine`; si también expira, solicita revisar DNS, proxy o reglas de salida para Docker Engine con el área de redes.

Si la base contiene tablas y no tiene una marca confiable, el CLI se detiene sin sembrar. Si el seed falla, no borres la base ni el volumen: conserva el diagnóstico y solicita revisión al equipo del sistema SER. Nunca uses `docker compose down -v`.

#### Aplicar cambios posteriores en `.env`

Edita `.env` con un editor seguro y conserva sus permisos restringidos. Después ejecuta:

```bash
sudo ser-qro-cli config apply
```

El comando valida Docker Compose y los recursos, compara la configuración con los contenedores y muestra **solo los nombres** de las variables modificadas y los servicios que recreará; nunca imprime los valores. Siempre advierte de la interrupción prevista y solicita confirmación. `--yes` omite la pregunta, pero no las advertencias. No uses `docker restart` esperando que vuelva a leer `.env`: Docker solo inyecta ese archivo al crear el contenedor.

Para cambios que afectan la aplicación web o PostgreSQL, el CLI activa el modo de mantenimiento de Laravel y espera 10 segundos antes de recrear servicios. Es una cortesía para que nuevas solicitudes reciban mantenimiento; **no garantiza** que una solicitud, transacción o trabajo que ya comenzó haya terminado. Si se recrea `app`, también se recrean `queue` y `nginx` cuando corresponda para que adopten el entorno y Nginx resuelva la dirección nueva de `app`. Los trabajos de cola se detienen con un periodo de espera basado en `QUEUE_TIMEOUT`; un trabajo que exceda su límite aún puede fallar. Los cambios de recursos o puerto recrean solo el servicio correspondiente. El CLI no elimina volúmenes.

Al finalizar, el CLI limpia la caché de configuración de Laravel, retira mantenimiento, confirma que los servicios estén activos y prueba `/login` en el puerto local configurado. Si la verificación falla, informa el estado y puede dejar SER en mantenimiento para evitar ofrecer una aplicación que no respondió. Revisa `docker compose ps` y `docker compose logs --tail=100 app queue nginx pgsql`; cuando el servicio esté sano, se puede retirar el mantenimiento con `docker compose exec -T app php artisan up`. Nunca uses `docker compose down -v`.

No todos los cambios se aplican con este comando:

- Correo, `APP_URL`, `ASSET_URL` y otras variables normales de Laravel se aplican recreando `app` y `queue`; habrá interrupción breve.
- `APP_HTTP_BIND` y `APP_HTTP_PORT` recrean Nginx. Ajustes de firewall o de un proxy externo se hacen por separado.
- Límites de CPU y memoria recrean el servicio cuyo límite cambió; si se recrea `app`, también `nginx` para que resuelva la dirección actual de la aplicación.
- `SER_VERSION` requiere `ser-qro-cli update vX.Y.Z`, que verifica las imágenes y genera el respaldo previo.
- Cambiar `DB_PASSWORD`, `DB_DATABASE`, `DB_USERNAME` o `DB_ADMIN_USERNAME` requiere coordinar las cuentas y la base existente; `config apply` se niega a aplicarlos.
- Cambiar `APP_KEY` puede impedir descifrar datos existentes y requiere un procedimiento de rotación; `config apply` se niega a aplicarlo.
- `SER_PROFILE`, los presupuestos y las rutas de respaldo que solo usa el CLI no recrean contenedores.

`APP_BASE_PATH` no forma parte de `.env.example`: los assets de Vite vienen compilados en la imagen del sistema SER y esa variable no se puede cambiar en runtime. Si aún aparece en un `.env` de una instalación anterior, se puede quitar; `config apply` la ignora y no requiere recrear contenedores. El contenedor existente puede conservar la variable obsoleta hasta que se recree por otro motivo, sin efecto en la aplicación. `ASSET_URL`, en cambio, sí es una variable de Laravel y puede aplicarse con `config apply`. En asignaciones vacías, coloca los comentarios en una línea aparte; por ejemplo `ASSET_URL=` debe quedar sin texto después del signo `=`, para evitar que Compose trate la explicación como valor.

Antes de abrir SER a usuarios, cambia desde la aplicación web la contraseña de la cuenta administradora inicial. Después verifica y programa respaldos:

```bash
ser-qro-cli doctor
ser-qro-cli backup run
ser-qro-cli backup install --frequency diaria --time 02:00 --retention-days 30
```

Por defecto, los respaldos quedan en el mismo disco de la VM y no protegen ante la pérdida de ese disco. Configura `SER_BACKUP_DIR` en `.env` solo cuando exista una ruta aprobada, montada en almacenamiento independiente y con permisos restringidos.

## 4. Opción C: Máquina virtual con SER ya desplegado

La VM se entrega con Rocky Linux 10.2, el CLI global, SER desplegado y el perfil `small`. No vuelvas a instalar el CLI, ejecutar `scaffold install` ni repetir `deploy`.

Al recibirla, revisa el estado:

```bash
ser-qro-cli version
ser-qro-cli resources show
ser-qro-cli doctor
```

Antes de abrir SER a usuarios, el personal municipal debe:

1. Confirmar con el equipo del sistema SER el valor correcto de `APP_URL`, la URL de Laravel que usarán los usuarios.
2. Verificar que `APP_KEY` esté resguardada y no cambiarla después de haber almacenado información cifrada.
3. Acordar la rotación de la contraseña de `ser_app` con el equipo del sistema SER; debe actualizarse en PostgreSQL y `.env` durante una ventana de mantenimiento.
4. Cambiar la contraseña de la cuenta administradora inicial de la aplicación web.
5. Verificar `ser-qro-cli doctor` y configurar respaldos con una copia en almacenamiento independiente.

Para habilitar UFEA o configurar el correo en esta VM, edita `/opt/web/ser-qro/.env` con un editor seguro y aplica los cambios con `sudo ser-qro-cli config apply`. No vuelvas a ejecutar el asistente de primer despliegue. El CLI mostrará los nombres de las variables y servicios afectados, advertirá de la interrupción y pedirá confirmación; nunca introduzcas tokens o contraseñas en argumentos de comandos.

Para cambiar la contraseña de `ser_app` en una VM ya desplegada, coordina ambas partes mientras SER está en mantenimiento y crea un respaldo previo:

```bash
cd /opt/web/ser-qro
ser-qro-cli backup run
docker compose exec -T app php artisan down --retry=60
docker compose exec pgsql psql --username=ser_bootstrap --dbname=postgres
```

En el prompt de `psql`, ejecuta `\password ser_app`, escribe dos veces la contraseña nueva y sal con `\q`. Edita `/opt/web/ser-qro/.env` con un editor seguro, actualiza `DB_PASSWORD` con el mismo valor y conserva el modo `0600`. Después:

```bash
docker compose up -d --force-recreate app queue
docker compose exec -T app php artisan up
ser-qro-cli doctor
```

Si falla la reconexión, deja SER en mantenimiento y solicita apoyo; no restaures ni elimines volúmenes como primera medida. La contraseña de `ser_bootstrap` se administra por separado: su rotación requiere coordinar PostgreSQL y el archivo protegido.

Cuando se requiera actualizar imágenes privadas o el CLI, prepara el asistente de credenciales y usa el PAT solo en el prompt:

```bash
ser-qro-cli auth setup --helper pass
ser-qro-cli auth login
ser-qro-cli auth status
ser-qro-cli self-update check
```

## 5. Administrar PostgreSQL con pgAdmin

El Compose base no publica el puerto de PostgreSQL; aplicación y base se comunican por la red interna de Docker. `scaffold install` deja el override `docker-compose.pgadmin.yaml` listo para publicar PostgreSQL **solo en `127.0.0.1`**, no en la red municipal.

Confirma que tu versión del CLI incluya el comando `db-access` con `ser-qro-cli db-access --help`; si no existe, actualiza el CLI a una versión aprobada antes de continuar.

#### Acceso SSH para el túnel (solo si pgAdmin está en otra computadora)

El túnel no es necesario para desplegar SER. Para pgAdmin remoto, la computadora debe poder conectarse por SSH al servidor. Si ya puedes iniciar una sesión con `ssh usuario@servidor`, omite esta preparación. Si no, necesitas acceso a la consola local o de la VM para instalar y habilitar el servicio:

```bash
# RHEL o Rocky Linux
sudo dnf install openssh-server
sudo systemctl enable --now sshd

# Ubuntu
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh
```

Usa una cuenta municipal individual, no `root`. La red o firewall debe permitir SSH (TCP 22) desde la computadora administrativa o la VPN municipal; no abras el acceso SSH a cualquier dirección de Internet. Si no sabes qué regla aplicar, consulta al responsable de la red antes de cambiar el firewall. Prueba primero que `ssh usuario@servidor` conecte correctamente.

Abre temporalmente el acceso:

```bash
ser-qro-cli db-access enable
```

Desde la computadora donde se ejecuta pgAdmin, abre un túnel SSH (el puerto por defecto es `15432`):

```bash
ssh -N -L 15432:127.0.0.1:15432 usuario@servidor
```

En pgAdmin usa servidor `localhost`, puerto `15432`, base `ser` y usuario `ser_bootstrap`. Obtén la contraseña del canal institucional seguro; no la pongas en la línea de comandos. Mantén activo el túnel mientras dure la sesión.

Al terminar, cierra pgAdmin y el túnel, y cierra la publicación:

```bash
ser-qro-cli db-access disable
```

Ambas acciones recrean únicamente PostgreSQL y no eliminan volúmenes, pero causan una interrupción breve. El CLI valida la configuración efectiva antes de recrear el servicio. Si el Compose principal todavía publica PostgreSQL en `0.0.0.0` u otra dirección, el CLI se niega a confirmar el cierre: esa publicación debe corregirse primero de forma coordinada. No uses `0.0.0.0` ni abras el puerto en el firewall. `DB_PORT=5432` es el puerto interno usado por Laravel, no el puerto externo de administración.

#### Cambiar la contraseña de `ser_bootstrap`

`ser_bootstrap` es la cuenta administrativa de PostgreSQL, distinta de `ser_app`. Para rotar su contraseña, cambia tanto la credencial del rol en PostgreSQL como el archivo protegido que usa el Compose al inicializar una base nueva. **Editar solo el archivo no cambia la contraseña del rol existente.** No la escribas en `.env`, en un argumento de comando ni en el historial del shell.

Hazlo en una ventana de mantenimiento acordada y conserva acceso al valor nuevo hasta terminar la verificación. Primero crea un respaldo y abre una sesión interactiva de `psql`:

```bash
sudo ser-qro-cli backup run
cd /opt/web/ser-qro
sudo docker compose exec -it pgsql psql --username=ser_bootstrap --dbname=postgres
```

En `psql`, ejecuta `\password ser_bootstrap`, captura la contraseña nueva dos veces en los prompts ocultos y sal con `\q`. Después actualiza el archivo protegido; el siguiente comando vuelve a pedirla dos veces sin mostrarla ni incluirla en argumentos:

```bash
sudo bash -c '
set -eu
file=/opt/web/ser-qro/secrets/ser_bootstrap_password
test -f "$file" && test ! -L "$file"
IFS= read -r -s -p "Nueva contraseña de ser_bootstrap: " first
printf "\n"
IFS= read -r -s -p "Confirma la nueva contraseña: " second
printf "\n"
if [ -z "$first" ] || [ "$first" != "$second" ]; then
  echo "Las entradas están vacías o no coinciden; no se modificó el archivo." >&2
  exit 1
fi
umask 077
temporary=$(mktemp "${file}.XXXXXX")
trap '\''rm -f -- "$temporary"'\'' EXIT
printf "%s" "$first" > "$temporary"
chown root:root "$temporary"
chmod 0600 "$temporary"
mv -f -- "$temporary" "$file"
unset first second
'
```

Verifica la contraseña nueva mediante una conexión TCP local que la solicite de forma interactiva:

```bash
sudo docker compose exec -it pgsql psql --host=127.0.0.1 --username=ser_bootstrap --dbname=postgres --password
```

Si conecta, sal con `\q` y actualiza también la contraseña guardada en el gestor de pgAdmin. No hace falta cambiar `DB_PASSWORD`, reiniciar PostgreSQL ni recrear los servicios: esta operación rota la credencial administrativa, no la del usuario que usa Laravel. Si la verificación falla, no cierres la sesión administrativa de recuperación; vuelve a alinear la contraseña del rol y el archivo antes de continuar.

## 6. Operación básica

Consulta la ayuda de cualquier comando con `ser-qro-cli COMANDO --help`; por ejemplo, `ser-qro-cli backup --help`. Los comandos se ejecutan en el servidor donde está el despliegue. Si `.env` y los archivos del despliegue son propiedad de `root`, ejecuta con `sudo` las operaciones que necesiten modificarlos.

### Consultar y ajustar recursos

Consulta los límites configurados, la suma de recursos asignados a los contenedores y si caben en la capacidad detectada del servidor:

```bash
ser-qro-cli resources show
ser-qro-cli doctor
```

`resources show` muestra los límites actuales y reporta errores si, por ejemplo, el presupuesto de memoria supera la RAM permitida después de reservar recursos para el sistema operativo. `doctor` vuelve a comprobar esta configuración junto con Docker, GHCR y los demás requisitos.

Para aplicar los valores del perfil `small`:

```bash
ser-qro-cli resources set small
ser-qro-cli resources show
```

El perfil `small` configura un presupuesto de 2 CPU y 3G de memoria para SER; aun así, el CLI valida que el servidor pueda reservar recursos para el sistema operativo. Si el perfil no cabe, la operación se rechaza y `.env` no se modifica.

Para ajustar valores individuales, ejecuta:

```bash
ser-qro-cli resources set custom
```

El CLI pregunta por el número de réplicas de la cola, presupuestos y límites de CPU/memoria por servicio y reservas del servidor. Escribe cada valor y presiona Enter; un Enter vacío conserva el valor actual. No guarda los cambios hasta validar la configuración completa. Si la valida, actualiza `.env` e indica el comando de Docker Compose para aplicar los nuevos límites. Aplicarlos recrea servicios y puede causar una interrupción; hazlo en una ventana autorizada. Después verifica con `ser-qro-cli resources show` y `ser-qro-cli doctor`.

### Respaldos y restauración

Para crear ahora un respaldo de la base de datos y de los archivos subidos, y para ver los respaldos disponibles:

```bash
ser-qro-cli backup run
ser-qro-cli backup list
```
**NOTA IMPORTANTE:**
`backup run` pone el sistema SER en mantenimiento durante la operación, esto quiere decir que los **usuarios no pueden acceder al sistema mientras se crea el respaldo**, por tanto se recomienda ejecutar este comando en una ventana autorizada de mantenimiento. En automático pone el sistema en mantenimiento, crea el respaldo y vuelve a iniciar la aplicación.

Al finalizar muestra la ruta creada, normalmente `/opt/web/ser-qro/backups/AAAAMMDD_HHMMSS/`; dentro quedan `ser_database.sql`, `ser_files.tar.gz` y metadatos de integridad. `backup list` muestra las rutas de los respaldos completos disponibles; si no hay ninguno, lo informa.

#### Frecuencia, hora e historial de la programación

`backup install` instala **una programación automática administrada por el CLI por servidor**. La opción predeterminada equivale a `--frequency diaria --time 02:00 --retention-days 30`. La hora se expresa en horario local y formato de 24 horas (`HH:MM`), desde `00:00` hasta `23:59`.

| Opción | Valores aceptados | Significado y valor predeterminado |
| --- | --- | --- |
| `--frequency` | `cada-hora` (`hourly`, `horaria`), `diaria` (`daily`), `semanal` (`weekly`) o `intervalo` (`interval`) | Tipo de programación. Predeterminado: `diaria`. |
| `--time` | `HH:MM` de 24 horas | Hora local para las frecuencias diaria, semanal e intervalo. Predeterminado: `02:00`. Con `cada-hora` se ejecuta al inicio de cada hora y este parámetro no aplica. |
| `--weekday` | `lun`, `mar`, `mie`, `jue`, `vie`, `sab`, `dom` (también `Mon`–`Sun`) | Día de ejecución si `--frequency semanal`; predeterminado: `lun`. Se ignora para otras frecuencias. |
| `--interval-days` | Entero mayor que cero | Solo para `--frequency intervalo`; días mínimos entre respaldos. Predeterminado: `1`. El programador comprueba diariamente a `--time` si ya venció el intervalo desde el último respaldo. |
| `--retention-days` | Entero mayor que cero | Conserva los respaldos de los últimos N días y elimina los anteriores cuando se crea un respaldo, manual o programado. Predeterminado: `30`. |

Ejemplos:

```bash
# Cada hora, al minuto 00
sudo ser-qro-cli backup install --frequency cada-hora --retention-days 30

# Cada domingo a las 03:30
sudo ser-qro-cli backup install --frequency semanal --weekday dom --time 03:30 --retention-days 60

# Cuando hayan transcurrido al menos 7 días desde el último respaldo
sudo ser-qro-cli backup install --frequency intervalo --interval-days 7 --time 02:00 --retention-days 30
```

Para programar un respaldo diario a las 02:00 y conservar los respaldos recientes, ejecuta como `root`:

```bash
sudo ser-qro-cli backup install --frequency diaria --time 02:00 --retention-days 30
```

El CLI instala la programación con systemd si está disponible; si no, usa cron. La salida confirma el mecanismo y la frecuencia instalados. La retención también se aplica al ejecutar `backup run`: los respaldos propios del CLI anteriores a los días indicados se eliminan. Puedes consultar `ser-qro-cli backup --help` para las opciones de frecuencia semanal o por intervalo.

Puedes consultar la programación activa según el mecanismo que use el servidor:

```bash
# systemd
sudo systemctl list-timers --all ser-qro-cli-backup.timer
sudo systemctl status ser-qro-cli-backup.timer

# cron (si el servidor no usa systemd)
sudo crontab -l | grep 'ser-qro-cli-backup'
```

Si el filtro de cron no muestra resultados, no hay una línea de respaldo del CLI con ese marcador.
`ser-qro-cli backup list` muestra respaldos guardados, **no** las tareas programadas. El CLI administra una programación por servidor; no permite mantener varias programaciones independientes. Volver a ejecutar `backup install` reemplaza la anterior (incluidos frecuencia, hora, intervalo y retención). Las tareas creadas manualmente fuera del CLI no quedan administradas por él y pueden provocar respaldos duplicados. Para quitar la programación administrada por el CLI:

```bash
sudo ser-qro-cli backup uninstall
```

Con frecuencia `intervalo`, systemd o cron invoca internamente `backup scheduled-run`. Este compara el tiempo desde el último respaldo registrado en `.last-backup`; si aún no transcurrió `--interval-days`, no crea otro. **No necesitas ejecutarlo manualmente**: `backup install` gestiona la programación. Para crear un respaldo fuera de calendario, usa `ser-qro-cli backup run`.

Para restaurar, primero ejecuta `backup list` y comprueba la ruta del archivo que necesitas. Por ejemplo, sustituye la carpeta de fecha por una mostrada en tu servidor:

```bash
ser-qro-cli restore db /opt/web/ser-qro/backups/AAAAMMDD_HHMMSS/ser_database.sql
ser-qro-cli restore files /opt/web/ser-qro/backups/AAAAMMDD_HHMMSS/ser_files.tar.gz
```

La restauración de base reemplaza los datos de la base; la de archivos reemplaza los archivos subidos. El CLI solicita confirmación, crea un respaldo de emergencia y pone SER en mantenimiento. Revisa cuidadosamente la ruta y el tipo de restauración antes de aceptar. Si la operación falla, sigue el diagnóstico del CLI; SER podría permanecer en mantenimiento.

`restore db` y `restore files` solo restauran los datos indicados; no cambian la versión de SER ni sus imágenes Docker.

El respaldo contiene base y archivos del sistema SER, no una copia completa de la VM. Guarda una segunda copia en almacenamiento independiente: el directorio predeterminado está en el mismo disco del sistema.

### Consultar versiones instaladas

Para consultar las versiones instaladas y disponibles del sistema SER y del CLI, utiliza el comando:

```bash
ser-qro-cli version
```

### Actualizar sistema SER

SER y el CLI tienen ciclos de actualización distintos. Para ver si hay una versión aprobada del sistema SER y sus imágenes están disponibles:

```bash
ser-qro-cli check-update
```

Usa una versión que el equipo del sistema SER haya aprobado, con formato `vMAJOR.MINOR.PATCH`; por ejemplo, `v1.2.3`. Sin argumento, `check-update` consulta la última versión estable. Este comando no cambia el despliegue: muestra la versión instalada y la disponible, y verifica las imágenes firmadas.

Para actualizar SER a una versión concreta:

```bash
ser-qro-cli update v1.2.3
```

La actualización verifica las firmas, crea un respaldo previo, pone SER en mantenimiento, descarga las imágenes aprobadas, recrea la aplicación y ejecuta explícitamente `php artisan migrate --force` contra la base existente. Los archivos de imagen que proporcionó el equipo de desarrollo confirman que el arranque no migra el esquema: el entrypoint prepara permisos, ejecuta `php artisan package:discover --ansi` y luego inicia el comando del contenedor (`php-fpm`); el workflow adjunto construye y publica imágenes, no ejecuta migraciones. En el flujo documentado, las migraciones las inicia `ser-qro-cli update`. `deploy` también migra durante el primer despliegue, pero solo si confirma que la base es nueva y permite el seed inicial. En cambio, `self-update` actualiza únicamente el ejecutable del CLI y `config apply` no ejecuta migraciones de esquema. Requiere acceso a GHCR y puede tardar varios minutos; acuerda una ventana de mantenimiento y no interrumpas el comando. No hay reversión automática si una migración ya modificó la base: el sistema puede quedar en mantenimiento y el CLI conserva el respaldo previo para diagnóstico o recuperación coordinada. Si no indicas versión, el CLI selecciona la última versión estable; para producción, especifica la versión aprobada.

### Actualizar herramienta de administración CLI

```bash
ser-qro-cli self-update check
sudo ser-qro-cli self-update install vX.Y.Z
```

`self-update check` muestra la versión instalada y la última publicada. `self-update install` descarga y verifica una versión del CLI antes de activarla; requiere `root` porque actualiza el ejecutable global. Consulta `ser-qro-cli self-update --help` para listar versiones instaladas o volver a una versión previa del CLI.

### Diagnóstico y soporte
Para generar un paquete de soporte que incluya información del sistema y los contenedores se puede usar el siguiente comando:

```bash
ser-qro-cli support-bundle
```
Este comando genera un archivo comprimido con información relevante para diagnóstico y soporte que se puede compartir con el equipo de soporte del sistema SER.

Para consultar la salida de los contenedores, usa `docker compose logs -f --tail=100 app` (o el servicio que falla). `storage/logs/.gitignore` es solo un marcador del directorio, no un archivo de log.

## 7. Seguridad y soporte

- Usa el PAT individual únicamente en el prompt de Docker. Nunca lo pongas en URLs, argumentos, variables persistentes, `.env`, imágenes, scripts, archivos de entrega ni historial de shell.
- La frase de paso de GPG protege la clave privada usada por `pass`; no la confundas con el PAT ni la compartas con soporte.
- Conserva `.env` y `secrets/ser_bootstrap_password` restringidos a root. Resguarda por separado `APP_KEY`, las contraseñas y las credenciales de recuperación.
- No habilites puertos, cambies el firewall, modifiques repositorios ni agregues usuarios al grupo `docker` sin autorización local.
- No publiques el puerto PostgreSQL en interfaces de red. Usa el comando `db-access` con túnel SSH para pgAdmin.
- No copies una VM ya desplegada para otro municipio sin preparar credenciales, claves y datos independientes.
- No consideres los respaldos del mismo disco una protección contra su pérdida.
- Para errores, ejecuta `ser-qro-cli doctor`; genera `ser-qro-cli support-bundle` y revísalo antes de enviarlo. Nunca envíes `.env`, PAT, frases GPG ni contraseñas.