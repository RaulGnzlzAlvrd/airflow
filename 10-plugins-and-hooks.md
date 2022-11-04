# Plugins y Hooks

## Introducción
Airflow tiene la facilidad de extender sus funcionalidades
de manera sencilla por medio de los Plugins.
Con estas herramientas tenemos la ventaja de añadir
funcionalidad o integrar una herramienta nueva a nuestros
pipelines de manera sencilla.

Airflow nos da la posibilidad de crear:
**Operators:** Ya sea extender uno existente o crear uno nuevo.

**Views:** Podemos agregar vistas a la interfaz web para 
monitorear alguna otra herramienta.

**Hooks:** Son conectores a alguna otra herramienta. Los
Operators hacen uso de los hooks para utilizar esas herramientas.

Por ejemplo podríamos crear un Hook para conectarnos a
un servidor de ElasticSearch y crear un Operador que use
este Hook para almacenar un dato, otro Operador para
recuperar otro datos, etc.

### Archivos
Todos los Hooks, Operators o Views deben de ir dentro de la carpeta
`plugins`. Hay que tener presente que cada que se agrega
un plugin nuevo se tiene que reiniciar Airflow ya que de
lo contrario no lo va a detectar. En caso de modificar un
plugin que ta tiene rastreado Airflow no es necesario
reiniciar.

### Extendiendo funcionalidad
Para el caso de las Views, se tiene que extender la
clase `AirflowPluginClass`. En el caso de los Hooks, se
tiene que extender la clase `BaseHook`, para los Operators
se extiende `BaseOperator` y solo hay que crear módulos
de python y colocarlos en la carpeta correcta.


## Ejemplo (ElasticSearch)
En este ejemplo se creará un Hook para interactuar con
un servidor de ElasticSearch, y un Operator que va a 
transferir datos desde Postgres a ElasticSearch.

### Hook
Dentro de la carpeta `plugins` creamos el siguiente
directorio y archivo, es donde estará el Hook:
```bash
mkdir -p plugins/elastic_search_plugin/hooks
cd plugins/elastic_search_plugin/hooks
touch elastic_hook.py
```

Creamos el Hook:
```python
# Todos los Hooks deben extender esta clase para compartir un mínimo de funcionalidad
from airflow.hook.base import BaseHook

# Se hace uso de esta biblioteca para realizar la conexión
from elasticsearch import Elasticsearch

class ElasticHook(BaseHook):
	def __init__(self, conn_id="elasticsearch_default", *args, **kwargs):
		"""
		conn_id: Es el nombre de la conexión definida dentro de la interfaz web
		"""
		super().__init__(*args, **kwargs) # Inicializamos superclase

		# Obtenemos la conexión definida en la interfaz web para acceder a sus atributos
		conn = self.get_connection(conn_id)

		# Opciones para inicializar una instancia de Elasticsearch
		conn_config = {}
		hosts = []
		fi conn.host:
			conn_config["host"] = conn.host.split(",")
		if conn.port:
			conn_config["port"] = int(conn.port)
		if conn.login:
			conn_config["http_auth"] = (conn.login, conn.password)

		# Creamos nuestra conexión por medio de la biblioteca
		self.es = Elasticsearch(hosts, **con_config)
		self.index = conn.schema


	## Agregamos funcionalidades al hook por medio de métodos

	def info(self):
		# Solo regresamos los detalles de la conexión por medio de la bilioteca
		returns self.es.info()

	def set_index(self, index):
		# Definimos el index que se va a usar
		self.index = index

	def add_doc(self, index, doc_type, doc):
		# Método que agrega un documento al index usando la conexión
		self.set_index(index)
		res = self.es.index(index=index, doc_type=doc_type, body=doc)
		return res
```

Como ejemplo para ver que esté funcionando bien el Hook, 
creamos un DAG de la forma usual:
```python
# Imports de siempre
from airflow import DAG
...

# Importamos el Hook desde el nombre de la carpeta dentro de plugins
# Se importa como un módulo usual de python
from elasticsearch_plugin.hooks.elastic_hook import ElasticHooks

def _print_es_info():
	# La usaremos con un PythonOperator
	# Aquí es donde usamos el Hook, pero es mejor usarlo dentro de
	# un operator
	hook = ElasticHook() 
	print(hook.info())

with DAG("elastic_dag", ...) as dag:

	# Creamos el task
	print_es_info = PythonOperator(
		task_id="print_es_info",
		python_callable=_print_es_info
	)
```

### Operator
Ahora crearemos el Operator para transferir datos de Postgres
a ElasticSearch.

Comenzamos creando el archivo:
```bash
mkdir -p plugins/elasticsearch_plugin/operators/
cd plugins/elasticsearch_plugin/operators/
touch postgres_to_elastic.py
```

Creamos el Operator:
```python
# Tenemos que extender BaseOperator
from airflow.models import BaseOperator

# Los hooks que vamos a utilizar. Como dijimos es mejor usarlos dentro de un Operator
from airflow.providers.postgres.hooks.postgres import PostgresHook
from elasticsearch_plugin.hooks.elastic_hook import ElasticHook # Este es nuestro Hook personalizado

# Para cerrar automáticamente las conexiones
from contextlib import closing
# Elasticsearch trabaja con objetos json
import json

class PostgresToElasticOperator(BaseOperator):
	"""
	Nuestro operador personalizado
	"""
	def __init__(self, sql, index,
				 postgres_conn_id="postgres_default",
				 elastic_conn_id="elasticsearch_default",
				 *args, **kwargs):
		"""
		sql: La consulta de donde se extraerán los datos
		index: En donde se almacenará el resultado de la consulta
		postgres_conn_id: El nombre de la conexión de postgres tal como la tenemos en la interfaz web
		elastic_conn_id: El nombre de la conexión de elasticsearch tal como la tenemos en la interfaz web
		"""
		super(PostgresToElasticOperator, self).__init__(*args, **kwargs)
		self.sql = sql
		self.index = index
		self.postgres_conn_id = postgres_conn_id
		self.elastic_conn_id = elastic_conn_id

	def execute(self, context):
		"""
		Este método siempre debe estar, es el que se va a ejecutar
		cuando utilicemos este operador
		"""
		# Usamos los hooks para conectarnos a herramientas externas
		es = ElasticHook(self.elastic_conn_id)
		pg = PostgresHook(self.postgres_conn_id)


		# Usamos la conexión a postgres
		with closing(pg.get_conn()) as conn:
			with closing(conn.cursor()) as cur:
				cur.itersize = 1000 # Leemos en chunks
				cur.execute(self.sql) # Ejecutamos la consulta que se pasó al operador
				for row in cur:
					doc = json.dumps(row, indent=2) # Convertimos a json
					
					# Almacenamos el resultado usando nuestro hook de elasticsearch
					es.add_doc(index=self.index, doc_type="external", doc=doc)
```

Cómo ejemplo de uso vamos a volver a usar el DAG de antes:
```python
...
from elasticsearch_plugin.hooks.elastic_hook import ElasticHooks
# Importamos nuestro operador personalizado
from elasticsearch_plugin.operators.postgres_to_elastic import PostgresToElasticOperator

def _print_es_info():
	...

with DAG("elastic_dag", ...) as dag:
	print_es_info = PythonOperator(...)

	# Definimos el Task usando nuestro operador
	connections_to_es = PostgresToElasticOperator(
		task_id="connections_to_es",
		sql="SELECT * FROM connections", # Esta sentencia la podemos cambiar
		index="connections" # En donde se va a almacenar el resultado
	)

	print_es_info >> connections_to_es
```
