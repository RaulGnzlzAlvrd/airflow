# Instalación de Airflow

> Se recomienda usar la imágen de docker. Una descripción detallada
está en la [sección 11](./11-docker.md).

## Ambiente virtual
Para facilitar el manejo de paquetes y tenerlos separados
de los paquetes del sistema, se recomienda usar un ambiente
virtual de python.

Para crear el ambiente: `python -m venv sandbox`

Para activar el ambiente: `source sandbox/bin/activate`
> Nota: debe ejecutarse desde el mismo directorio que el
comando de creción de ambiente.

## Instalación de paquetes
Teniendo el ambiente virtual activado, instalar el
paquete `wheel`: `pip install wheel`

En este [gist](https://gist.github.com/marclamberti/742efaef5b2d94f44666b0aec020be7c#file-airflow-version)
hay que revisar el archivo `Airflow Version` que tiene la versión
de airflow que se tiene que instalar. También viene el archivo
`constraint.txt` con las dependencias a instalar para que no haya errores.

Así el comando a ejecutar sería:
```
pip install apache-airflow==[version] --constraint https://gist.githubusercontent.com/marclamberti/742efaef5b2d94f44666b0aec020be7c/raw/21c88601337250b6fd93f1adceb55282fb07b7ed/constraint.txt
```
> Nota: el link para el constraint se obtiene de clickar en `Raw`
en el gist de `constraint.txt`

## Configuración inicial
Antes de poder ejecutar airflow se tiene que crear los
archivos de configuración básica e incializar el
metastore con el comando: `airflow db init`

Luego se crea el usuario administrador para poder iniciar
sesión en la interfaz web, tiene el nombre de usuario `admin`
y la contraseña `admin`:
```
airflow users create -u admin \
					 -p admin \
					 -f first_name \
					 -l last_name \
					 -r Admin \
					 -e admin@airflow.com
```

## Iniciar scheduler e interfaz
Con toda la configuración anterior hecha, si se quiere inciar
el scheduler y la interfaz web, hay que ejecutar:
1. `airflow scheduler`
2. `airflow webserver`

Al ejecutar cada uno de los comandos se van a manterner en
la consola actual y se cierran cuando termina la sesión en
la terminal o pulsando `CTRL+C`.

TODO: Agregar cómo mantenerlos ejecutando en segundo plano.

La interfaz se ejecuta por defecto en el puerto `8080`.
Desde el navegador se visita: `localhost:8080`.
