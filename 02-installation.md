# Instalación de Airflow

> Se recomienda usar la imágen de docker. Una descripción detallada
está en la [sección 11](./11-docker.md).

## Ambiente virtual
Para facilitar el manejo de paquetes y tenerlos separados
de los paquetes del sistema, se recomienda usar un ambiente
virtual de python.

Para crear el ambiente ejecutar el siguiente comando: 
```
python -m venv sandbox
```

Una vez creado el ambiente hay que activarlo, para ello ejecutar el comando: 
```
source sandbox/bin/activate
```

> Nota: El comando anterior debe ejecutarse desde el mismo directorio que el
comando de creción de ambiente.

## Instalación de paquetes
Teniendo el ambiente virtual activado, instalar el paquete `wheel`, que
es necesario para ejecutar Airflow: 
```
pip install wheel
```

Luego instalar el paquete `apache-airflow` con el siguiente comando:
```
pip install apache-airflow
```

## Configuración inicial
Antes de poder ejecutar airflow se tiene que crear los
archivos de configuración básica e incializar el
metastore. Para ello ejecutar el comando: 
```
airflow db init
```

> El comando anterior creará la carpeta principal de Airflow en
> el directorio `~/airflow`.

Luego se necesita crear el usuario administrador para poder iniciar
sesión en la interfaz web. 
En el siguiente ejemplo se elige el nombre de usuario `admin` (con la opción `-u`)
y la contraseña `admin` (con la opción `-p`), pero puedes escoger el usuario y
contraseña de tu preferencia:
```
airflow users create -u admin \
					 -p admin \
					 -f first_name \
					 -l last_name \
					 -r Admin \
					 -e admin@airflow.com
```

## Iniciar scheduler e interfaz
Con toda la configuración anterior hecha, ahora se tiene que iniciar
el scheduler y la interfaz web. Para ello hay que mantener ejecutandose
el comando:
```
airflow scheduler
```

Una vez que ya se está ejecutando el comando anterior hay que abrir una
terminal nueva y asegurarse de tener activado el ambiente. Para inicializar
la interfaz web ejecutar el comando:
```
airflow webserver
```

> Para detener los dos comandos anteriores solo hay que cerrar la terminal o
> pulsar `CTRL+C`.

La interfaz web se ejecutará por defecto en el puerto `8080`. Así que
para acceder a ella visitar desde el navegador: [localhost:8080](http://localhost:8080).
