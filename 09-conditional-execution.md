# Ejecución condicional de tasks
Hay situaciones en las que se quiere ejecutar un task u
otro dependiendo de una condición que se de en los tasks
anteriores. Airflow soluciona este problema con el uso de
el operador `BranchPythonOperator`.

Nuevamente vamos a usar el archivo `xcom_dag.py` como
ejemplo. Queremos ejecutar el task `accurate` si alguno de
los modelos tuvo un accuracy mayor a 5, y queremos ejecutar
el task `inaccurate` si todos los modelos fueron menores o
iguales a 5.
```python
...
# Importamos el operador para tomar decisiones
from airflow.operators.python import BranchPythonOperator
from airflow.operators.dummy import DummyOperator

...

def _training_model(task_instance):
	...

# La función debe regresar el id del task que se va a
# ejecutar, o la lista de tasks que se van a ejecutar.
def _choose_best_model(task_instance):
    print('choose best model')
    accuracies = task_instance.xcom_pull(key="model_accuracy", task_ids=[
        "processing_tasks.training_model_a",
        "processing_tasks.training_model_b",
        "processing_tasks.training_model_c"
    ])
    print(accuracies)
	for accuracy in accuracies:
        if accuracy > 5:
            return "accurate"
    return "inaccurate"

with DAG(...) as dag:
    downloading_data = BashOperator(...)

    with TaskGroup('processing_tasks') as processing_tasks:
        training_model_a = PythonOperator(...)
        training_model_b = PythonOperator(...)
        training_model_c = PythonOperator(...)

	# Usamos el operador Branch
    choose_model = BranchPythonOperator(
        task_id='choose_model',
        python_callable=_choose_best_model #Función que regrese el id del task elegido
    )

	# Los tasks a ejecutar, se usa el operador Dummy solo de prueba
	accurate = DummyOperator(
        task_id="accurate"
    )

    inaccurate = DummyOperator(
        task_id="inaccurate"
    )

    downloading_data >> processing_tasks >> choose_model
	# Solo se ejecutará una dependiendo del resultado de choose_model
	choose_model >> [accurate, inaccurate]
```

Se usa el operador `BranchPythonOperator` que recibe como parámetros 
su id y una función de python que debe regresar un id o una lista de ids
de las tasks elegidas a ejecutar a continuación.

## Trigger Rules
Por defecto Airflow ejecuta un Task solo si todas las tasks de las
que depende se ejecutaron con éxito, pero podemos cambiar el comportamiento
para que por ejemplo, se ejecute un task solo si al menos uno de los que
depende falló, o se ejecute un task si la mayoría de los tasks de los que
depende tuvieron éxito.

Hay 9 tipos de Trigger Rules, algunas de las más importantes son:
- `all_success`: Es el comportamiento por defecto de Airflow
- `all_failed`: Se ejecuta el task si todas sus dependencias fallaron
- `all_done`: Se ejecuta el task si todas sus dependencias terminan, no importa si fueron fallidas.
- `one_sucess`: Se ejecuta el task en cuanto una de sus dependencias tiene éxito.
- `one_failed`: Se ejecuta el task en cuanto una de sus dependencias falla.
- `none_failed`: Se ejecuta el task siempre y cuando todas sus dependencias tienen éxito o son omitidas. Si alguna tiene un fallo, entonces no se ejecuta.

### Ejemplo
Usando nuevamente el ejemplo de `xcom_dag.py`. Si queremos añadir un nuevo
Task al final del pipeline, este no se va a ejecutar ya que por defecto solo
se ejecuta si los tasks de los que dependa (`inaccurate` y `accurate`) se
ejecutan con éxito. Cómo siempre va a pasar que uno se ejecute y el otro sea
omitido, podemos cambiar su trigger rule para que se comporte como queremos:
```python
...

with DAG(...) as dag:
    downloading_data = BashOperator(...)

    with TaskGroup('processing_tasks') as processing_tasks:
		...

    choose_model = BranchPythonOperator(...)
	accurate = DummyOperator(...)
    inaccurate = DummyOperator(...)

	# El task que se debe ejecutar siempre al final
	storing = DummyOperator(
		task_id="storing",
		trigger_rule="none_failed" # Con esta regla decimos que se ejecute aunque no hayan tenido éxito todas sus dependencias.
	)

    downloading_data >> processing_tasks >> choose_model
	choose_model >> [accurate, inaccurate] >> storing #Lo agregamos al final
```
