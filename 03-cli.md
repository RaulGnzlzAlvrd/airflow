# Interfaz de línea de comandos (CLI)

Airflow cuenta con una interfáz de línea de comandos (CLI por
sus siglas en ingles) con la cuál se puede administrar y controlar
todos los componentes de Airflow.

## Algunos comandos útiles

**Incialización de metastore**, este comando solo se debería ejecutar
una vez:
```
airflow db init
```

**Creación de usuario**, se pueden cambiar los datos a conveniencia:
```
airflow users create -u user_name \
					 -p password \
					 -f first_name \
					 -l last_name \
					 -r Admin \
					 -e user@example.com
```

**Iniciar el scheduler**:
```
airflow scheduler
```

**Iniciar la interfaz web**:
```
airflow webserver
```

**Actualizar la versión del metastore**, si se actualiza a una versión
nueva de airflow, se puede mantener el mismo metastore y solo actualizarlo
con:
```
airflow db upgrade
```

**Reconstruir el metastore**. Se elimina todo lo que había en el
metastore y se vuelven a generar los archivos de configuración inicial:
```
airflow db reset
```

**Mostrar DAGs disponibles**. Se obtiene una tabla con todos los DAGs
en el sistema, su `dag_id`, su ubicación y si se está ejecutando o no:
```
airflow dags list
```

**Mostrar Tasks de un DAG**. Se obtiene un listado de las tareas que
contiene el DAG identificado por `dag_id`:
```
airflow tasks list [dag_id]
```

**Ejecutar un DAG**. Se programa la ejecución del DAG identificado por
`dag_id` para la fecha puesta en `date`:
```
airflow dags trigger -e [date] [dat_id]
```

## Obtener ayuda
Para ver una descripción de los comandos y grupos de comandos
que tiene airflow:
```
airflow --help
```

Para ver una descripción de las opciones de algún comando o
grupo:
```
airflow [comando_or_grupo] --help
```

Por ejemplo si se quieren ver las opciones del comando `webserver`:
```
airflow webserver --help
```
