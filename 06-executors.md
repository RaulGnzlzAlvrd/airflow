## Variables de configuración

Airflow ya tiene configuraciones por defecto que se encuentra en el
archivo `airflow.cfg`. Las opciones están divididas en secciones.

Para revisar solo alguna de las configuraciones se puede usar la
línea de comandos:
```bash
$ airflow config get-value [section] [option]
```

Por ejemplo para saber cuál es el tipo de Executor que usa por defecto
Airflow: `airflow config get-value core executor`. La opción executor
está en la sección core.

## Executors

El Executor define la forma en que se ejecutan las tasks de nuestro DAG.
Por defecto Airflow usa el `SequentialExecutor`, lo que quiere decir que
las tasks se van a ejecutar de forma lineal, aún cuando hay tasks que
se pueden ejecutar en paralelo.

Por ejemplo, en el siguiente DAG tenemos que `task_2` y `task_3` se
pueden ejecutar en paralelo:

![](./imgs/parallel_dag.png)

Pero en realidad se van a ejecutar de forma secuencial, de hecho podría
ejecutarse primero el 3 y luego el 2 o viceversa:

![](./imgs/parallel_gant.png)

## LocalExecutor
El `LocalExecutor` define que las tasks se pueden ejecutar de forma
paralela, pero para ello se necesita otra base de datos.
Por defecto Airflow funciona usando SQLite, pero no permite hacer consultas
concurrentes, lo cuál es necesario para que funcione el `LocalExecutor`.

Necesitaremos de otro manejador de bases de datos, en este caso usaremos
Postgres.

### Instalación de paquetes necesarios
Instrucciones para Ubuntu:
1. Instalar el paquete: 
```bash
$ sudo apt update && sudo apt install postgresql
```
2. Acceder al interprete de postgres como administrador de la base:
```bash
$ sudo -u postgres psql
```
3. Cambiar la contraseña del administrador de la base:
```bash
> ALTER USER postgres PASSWORD 'postgres';
> \q
```
4. Instalar paquetes de postgres para airflow:
```bash
$ pip install 'apache-airflow[postgres]'
```

### Configuración de Airflow
1. Abrir el archivo `airflow.cfg`
2. Cambiar la opción del metastore:
```
sql_alchemy_conn=postgresql+psycopg2://[user]:[password]@[host]/[database]
```
En nuestro caso sería:
```
sql_alchemy_conn=postgresql+psycopg2://postgres:postgres@localhost/postgres
```
3. Verificar la conexión:
```bash
$ airflow db check
```
4. Cambiar la opción del Executor:
```
executor=LocalExecutor
```
5. Inicializar el metastore para usarlo con postgres:
```bash
$ airflow db init
```
6. Creación de primer usuario:
```bash
$ airflow users create -u admin \
						-p admin \
					 	-f first_name \
					 	-l last_name \
					 	-r Admin \
					 	-e admin@airflow.com
```
> En caso de estar ejecutando el scheduler y la interfaz web hay que detenerlos 
antes del paso 5 y volver a iniciarlos luego del paso 6.

Si volvemos a ejecutar el DAG, ahora en la vista de Gantt confirmamos que
el `task_2` y el `task_3` se ejecutaron de forma paralela:

![](./imgs/parallel_local_executor.png)

## Celery Executor
Permite ejecutar tasks en una arquitectura multi nodo.
Como base de datos para el metastore se puede usar postgres, en la sección
anterior se explicó cómo configurarlo.

1. Instalar el paquete Celery:
```bash
$ pip install apache-airflow-providers-celery==2.0.0
```

Cómo se explicó en el archivo de [instalación](./02-instalacion.md), esta
arquitectura necesita de un queue. En ese caso se usa [Redis](https://redis.io/)
2. Instalar Redis
```bash
$ sudo apt update
$ sudo apt install redis-server
```

3. Configurar redis:
```bash
$ sudo nano /etc/redis/redis.conf
```

4. Cambiar la opción de `supervised` a `systemd`:
```
supervised systemd
```

5. Guardar y cerrar el archivo
6. Reiniciar el servidor de redis:
```bash
$ sudo systemctl restart redis.service
```
7. Revisar que se esté ejecutando correctamente 
```bash
$ systemctl status redis.service
```

8. Cambiar las siguientes opciones en la configuración de airflow:
```
executor = CeleryExecutor
...
broker_url = redis://localhos:6379/0
...
result_backend = db+postgres://postgres:postgres@localhost/postgres
```

9. Instalar los paquetes para redis
```bash
$ pip install 'apache-airflow[redis]'
```

10. Iniciar Flower:
```bash
$ airflow celery flower
```
11. Iniciar workers en cada máquina
```bash
$ airflow celery worker
```

> Cada worker debe tener la misma configuración de `broker_url` y `result_backend`.
Además debe tener los providers necesarios apra ejecutar las tasks.

## Casos de uso para cada Executor
- SequentialExecutor: Cuando se quiere debuguear, entonces los ejecutamos uno después del otro
- LocalExecutor: Para usar concurrencia pero en una sola máquina
- CeleryExecutor: Cuando se quiere concurrencia y una gran cantidad de tasks ejecutandose al mismo tiempo

## Opciones de paralelismo
Con airlfow se puede limitar y cambiar distintas opciones para controlar
el paralelismo, tales como: el número de tasks que se ejecutan
en toda la instancia, o el número de DAGs que se ejecutan al mismo tiempo.

- `paralelism`: el número máximo de tasks que se pueden ejecutar de forma paralela.
- `dag_concurrency`: El número máximo de tasks que se pueden ejecutar para un solo DAG de forma paralela en todos sus DAG runs.
> También se puede controlar para un DAG específico al momento de crearlo, usando el atributo `concurrency`
- `max_active_runs_per_dag`: El número de DAGs que se pueden estár ejecutando al mismo tiempo
> También se puede controlar para un DAG específico al momento de crearlo, usando el atributo `max_active_runs`

Lo recomendable es usar las opciones del archivo `airflow.cfg` para ajustarse a los recursos de cada
dispositivo (o worker). Y usar los atributos de la clase DAG, cuando se quiere ajustar al comportamiento
de el DAG en específico (por ejemplo cuando un DAG run depende del anterior).
