# Instalación con Docker

En la segunda guía se explicó cómo configurar una máquina
virtual para ejecutar todo el stack de airflow, que incluye
el metastore, el scheduler, el webserver, y en caso de usar
la arquitectura Celery, también incluye los workers, y flower.
En esta sección se explica cómo instalar el stack de airlfow
usando Docker.

> No se explican las bases de Docker ni Docker-compose.
Para ello consultar la
[documentación de Docker](https://docs.docker.com/get-started/overview/).

> Se da por hecho que ya se tiene instalado Docker y
Docker-compose en la máquina. Consultar en la documentación cómo
instalar para su sistema operativo.

> Para una versión más actualizada consultar la documentación
[Running Airflow in Docker](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html).

## Configuración
Obtener el archivo docker compose para airflow:
```bash
$ curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.3.2/docker-compose.yaml'
```
> Consultar la
[documentación](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html#docker-compose-yaml)
para obtener el archivo más reciente.

Este archivo contiene toda la configuración para ejecutar
cada uno de los componentes de airflow como un contenedor
diferente. El archivo está configurado por defecto para
usar la arquitectura Celery.

## Configuración común
Primero que nada se definen las configuraciones que van a ser
comunes para cada imágen dentro de nuestro stack, para ello se
definen las siguientes configuraciones dentro de la
sección `x-airflow-common`:

### Versión de la imagen base
En la sección de `image` se puede configurar la versión de
airflow que se quiere usar:
```yaml
  image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.3.2}
```

### Variables de entorno
En la sección del archivo `docker-compose.yaml` tenemos las
variables de entorno de los contenedores, se componen de 3 partes:
`AIRFLOW` para indicar que son variables de configuración de airflow,
seguida de dos guiones bajos `__`, luego la sección en la configuración,
por ejemplo `CORE`, nuevamente dos guiones bajos, y finalmente la
opción de configuración, por ejemplo `LOAD_EXAMPLES`.

Algunas configuraciones de ejemplo son las siguientes:
```yaml
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__LOAD_EXAMPLES: 'true'
```

### Volumenes
Para poder agregar nuestros DAGs o plugins a airflow se usan los
volúmenes de Docker, estos se configurar en la sección de `volumes`:
```yaml
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins
```

Para poder hacer uso de ellos, tenemos que crear los directorios y 
configurar permisos mediante variables de entorno:
```bash
$ mkdir dags/ logs/ plugins/
$ echo -e "AIRFLOW_UID=$(id -u)" > .env
```

### Dependencias entre contenedores
Para que se pueda ejecutar en conjunto todos los contenedores
necesitamos que se ejecuten en cierto orden, esto se define en
la sección `depends_on`. En este caso como es la configuración
para la arquitectura Celery, necesitamos de una base de datos
y un servicio de queues como redis:
```yaml
  depends_on:
    &airflow-common-depends-on
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy
```

## Servicios
En la sección de servicios, definida por `services`, se definen
los contenedores a usar. Cada contenedor es un componente del
stack de airflow, por ejemplo uno es el metastore, otro el webserver,
etc.
En estas secciónes se puede configurar en qué puertos va a estar
ofreciendo sus servicios cada contenedor, por ejemplo el webserver
se estará ejecutando en el puerto `8080`.

## Ejecución del stack (Celery)
Para ejecutar todo el stack (por defecto está configurado con
la arquitectura Celery), se ejecuta el comando:
```bash
$ docker-compose up airflow-init
$ docker-compose up -d
```

Para detener y limpiar todo:
```bash
$ docker-compose down --volumes --rmi all
```
> Si solo se quiere desactivar y no limpiar las imágenes y
volúmenes usar: `docker-compose down`

## CLI
Para poder ejecutar los comandos que ofrece airflow se tienen
que ejecutar desde algún contenedor con nombre `airflow-*` por
ejemplo:
```bash
$ docker-compose run airflow-worker airflow info
```

## Local Executor
Por defecto estamos usando el Celery Executor, pero si queremos
simplemente el Local Executor tenemos que realizar algunas modificaciones
al archivo `docker-compose.yaml` ya que en el Local Executor no
necesitamos de los workers o de un queue.

1. Cambiar el Executor en las variables de entorno:
```yaml
# docker-compose.yaml
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
```

2. Quitar o comentar las opciones `AIRFLOW__CELERY__*`:
```yaml
    # AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://airflow:airflow@postgres/airflow
    # AIRFLOW__CELERY__BROKER_URL: redis://:@redis:6379/0
```

3. Quitar o comentar la dependencia en redis:
```bash
  depends_on:
    &airflow-common-depends-on
    # redis:
    #   condition: service_healthy
```

4. Quitar o comentar los servicios `redis`, `worker`, `flower`:
```yaml
services:
  ...

  # redis:
  #   image: redis:latest
  #   expose:
  #     - 6379
  #   healthcheck:
  #     test: ["CMD", "redis-cli", "ping"]
  #     interval: 5s
  #     timeout: 30s
  #     retries: 50
  #   restart: always

  ...

  # airflow-worker:
  #   <<: *airflow-common
  #   command: celery worker
  #   healthcheck:
  #     test:
  #       - "CMD-SHELL"
  #       - 'celery --app airflow.executors.celery_executor.app inspect ping -d "celery@$${HOSTNAME}"'
  #     interval: 10s
  #     timeout: 10s
  #     retries: 5
  #   environment:
  #     <<: *airflow-common-env
  #     # Required to handle warm shutdown of the celery workers properly
  #     # See https://airflow.apache.org/docs/docker-stack/entrypoint.html#signal-propagation
  #     DUMB_INIT_SETSID: "0"
  #   restart: always
  #   depends_on:
  #     <<: *airflow-common-depends-on
  #     airflow-init:
  #       condition: service_completed_successfully

  ...

  # flower:
  #   <<: *airflow-common
  #   command: celery flower
  #   profiles:
  #     - flower
  #   ports:
  #     - 5555:5555
  #   healthcheck:
  #     test: ["CMD", "curl", "--fail", "http://localhost:5555/"]
  #     interval: 10s
  #     timeout: 10s
  #     retries: 5
  #   restart: always
  #   depends_on:
  #     <<: *airflow-common-depends-on
  #     airflow-init:
  #       condition: service_completed_successfully
```

Ejecutar con:
```bash
$ docker-compose up airflow-init
$ docker-compose up -d
```


## Customizar la imagen
Si se quieren usar bibliotecas concretas de python o algún
proveedor de airflow consultar la
[documentación](https://airflow.apache.org/docs/docker-stack/build.html)
