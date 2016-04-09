# Un stack ELK (Elasticsearch - Logstash - Kibana) con Docker

Este README describe un conjunto de comandos para construir un stack ELK todo dentro
una misma máquina host. Utiliza logspout para recolectar los logs y enviarlos a un contendor logstash que actúa como sumidero y que finamelte los reenvía a un contenedor elasticsearch. Un contenedor Kibana visualiza los datos.
Para probar utilizamos un contenedor logstest que envía una salida por stdout.

También hay definida un fichero elk.yml para levantar la infraestructura con docker-compose.

## Creación de la máquina virtual para ELK

``` bash
  $ docker-machine create -d virtualbox elk
```

## Selección la máquina ELK
``` bash
  $ eval $(docker-machine env elk)
```

## Creación del contenedor Elasticsearch (E--)
``` bash
  $ docker run --name="elasticsearch" --rm -it -e LOGSPOUT="ignore" elasticsearch
```

La variable de entorno `$LOGSPOUT` indica al recolector de logs `logspout`
que no incluya los logs de este contenedor.

Este contenedor expone el puerto 9200 y 9300. El puerto 9200 lo usa Kibana y el
puerto 9300 Logstash.

## Creación del contenedor Kibana (--K)
``` bash
  $ docker run --name="kibana" --rm -it --link elasticsearch:elasticsearch \
    -e LOGSPOUT="ignore" -e ELASTICSEARCH_URL="http://elasticsearch:9200" \
    -p 5601:5601 kibana
```

La conexión con el contenedor de Elasticsearch es realizada en primer lugar
con un link al contenedor "elasticsearch" y luego con la variable de
entorno `$ELASTICSEARCH_URL` que actúa como endpoint.

El contenedor expone el puerto 56001 para consulta web.

## Creación del contenedor Logstash (-L-)
``` bash
  $ docker run --name="logstash" --rm -it --link elasticsearch:elasticsearch \
    -v "$PWD":/config-dir -e LOGSPOUTignore" \
    logstash logstash -f /config-dir/logstash.conf
```

Este contenedor expone el puerto 5000 para la recepción de logs. En nuesto caso
los logs los recibirá del contenedor con "logspout". Utiliza el fichero de
configuración logstash.conf. Este fichero describe el formato de los logs de
entrada, filtros para los logs de entrada, y la salida o reenvío al contenedor
"elasticsearch". Por defecto utiliza el puerto 9300 hacia "elasticsearch".

El fichero `logstash.conf` contiene un filtro para cambiar el punto "." en el nombre
de los keys por un guion bajo "_", es necesario porque elasticsearch a partir
de la versión 2.0 no admite keys con un punto. Ver [Field name cannot contain ‘.’](https://discuss.elastic.co/t/field-name-cannot-contain/33251/15). Existe un
filtro para esta situación. Ver [de_dot](https://www.elastic.co/guide/en/logstash/master/plugins-filters-de_dot.html)

## Creación del contenedor Logspout

Este es el cuarto contenedor del Stack y demoniza el servicio encargado de
sacar los logs Docker de la máquina anfitrión que no están marcados como
"ignore". Actúa como un recolector local de logs hacia el servicio "logstash".

``` bash
  $ docker run --name="logspout" --rm -it -v /var/run/docker.sock:/tmp/docker.sock \
    -p 8000:80 --link logstash:logstash \
    amouat/logspout-logstash logstash://logstash:5000
```

Además enlaza el puerto 8000 de la máquina  con el puerto 80 para realizar
consulta de los logs via http.

## Creación de un contendor de prueba

Este contenedor envía un mensaje a stdout que puede ser consultado con Kibana.

``` bash
  $ docker run --name="logstest" --rm debian \
    sh -c 'echo "Hello World!"'
```

## Interfaz Web de Kibana

La dirección de ip de la máquina elk resulta de:

``` bash
  $ docker-machine ip elk
```

Luego es accesible con el navegador con esta ip y el puerto  6001 desde la máquina hosts.

## docker-compose

La infraestrucutra puede desplegarse con `docker-compose` y comprobarse igual

``` bash
  $ docker-compose -f elk.yml up -d
```
