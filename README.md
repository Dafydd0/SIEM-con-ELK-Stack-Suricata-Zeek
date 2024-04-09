# SIEM con ELK Stack Suricata y Zeek

Este repositorio ha sido creado con la finalidad de hacer más sencilla la configuración de los IDS Suricata y Zeek junto con la pila ELK para la creación de SIEM. Se describirá la configuración necesaria para desplegar tanto la pila ELK como los IDS, además de los pasos para configurar las reglas personalizadas para detectar el malware WannaCry. Los archivos de configuración que son necesarios modificar se encuentran a disposición del usuario aunque es importante destacar que los valores como la dirección IP o la interfaz de red pueden cambiar dependiendo de la configuración de cada equipo.

## Configuración Suricata

Modificar el fichero suricata.yaml, en este, se debe cambiar el nombre de la interfaz de red donde estará escuchando Suricata.

```
sudo nano /etc/suricata/suricata.yaml
```

Alrededor de la línea 580 se encontrará el siguiente código donde se debe modificar el nombre de la interfaz:

```
# Linux high speed capture support
af-packet:
  - interface: eth0
    # Number of receive threads. "auto" uses the number of cores
    #threads: auto
    # Default clusterid. AF_PACKET will load balance packets based on flow.
    cluster-id: 99
. . .
```

Para que Suricata aplique los cambios se debe ejecutar la siguiente instrucción:

```
sudo suricata-update
```

Para validar que todo se ha realizado de manera correcta se puede ejecutar el siguiente comando:

```
sudo suricata -T -c /etc/suricata/suricata.yaml -v
```

## Configuración Elasticsearch

Se debe modificar el archivo *elasticsearch.yml*, en el debemos añadir la IP local y la IP privada en caso de que deseemos que Elasticsearch opere fuera de la red local.

```
sudo nano /etc/elasticsearch/elasticsearch.yml
```

La configuración de la red se encuentra sobre la línea 55:

```
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
#network.host: 192.168.0.1
network.bind_host: ["127.0.0.1", "your_private_ip"]
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
```
Al final del archivo se deben añadir las siguientes instrucciones:

```
. . .
discovery.type: single-node
xpack.security.enabled: true
```

Ahora debemos configurar la seguridad de la pila ELK, para ello debemos generear una serie de claves con las siguientes instrucciones.

```
cd /usr/share/elasticsearch/bin
sudo ./elasticsearch-setup-passwords auto
```
Las contraseñas generadas aparecerán y tendrán el siguiente formato.

```
Changed password for user apm_system
PASSWORD apm_system = eWqzd0asAmxZ0gcJpOvn

Changed password for user kibana_system
PASSWORD kibana_system = 1HLVxfqZMd7aFQS6Uabl

Changed password for user kibana
PASSWORD kibana = 1HLVxfqZMd7aFQS6Uabl

Changed password for user logstash_system
PASSWORD logstash_system = wUjY59H91WGvGaN8uFLc

Changed password for user beats_system
PASSWORD beats_system = 2p81hIdAzWKknhzA992m

Changed password for user remote_monitoring_user
PASSWORD remote_monitoring_user = 85HF85Fl6cPslJlA8wPG

Changed password for user elastic
PASSWORD elastic = 6kNbsxQGYZ2EQJiqJpgl
```

## Configuración Kibana

En primer lugar debemos generar las claves de encriptación.

```
cd /usr/share/kibana/bin/
sudo ./kibana-encryption-keys generate -q
```

En la terminal aparecerá algo parecido a esto.

```
xpack.encryptedSavedObjects.encryptionKey: 66fbd85ceb3cba51c0e939fb2526f585
xpack.reporting.encryptionKey: 9358f4bc7189ae0ade1b8deeec7f38ef
xpack.security.encryptionKey: 8f847a594e4a813c4187fa93c884e92b
```

Tras esto debemos modificar el archivo kibana.yml y al final de este añadir las claves generadas en el paso previo.

```
sudo nano /etc/kibana/kibana.yml
```

Adicionalmente dentro de este archivo, debemos modificar la red donde operará Kibana, en nuestro caso es la 10.0.2.15.

```
# Kibana is served by a back end server. This setting specifies the port to use.
#server.port: 5601

# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
#server.host: "localhost"
server.host: "your_private_ip"
```

Ahora debemos configurar las credenciales de Kibana.

```
sudo ./kibana-keystore add elasticsearch.username
```
```
sudo ./kibana-keystore add elasticsearch.password
```
El usuario es *kibana_system* y la contraseña es la generada en el paso de de configuración de elasticsearch, aque cuyo nombre de usuario sea *kibana_system*.

## Configuración de Filebeat

Debemos modificar el archivo *filebeat.yml*.

```
sudo nano /etc/filebeat/filebeat.yml
```

Debemos añadir la IP de Kibana host, 10.0.2.15 en nuestro caso.

```
. . .
# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"
  host: "your_private_ip:5601"
. . .
```

Adicionalmente en este mismo documento, para terminar la configuración y que nos podamos conectar a Kibana, alrededor de la línea 130 debemos modificar lo siguiente.

```
output.elasticsearch:
  # Array of hosts to connect to.
  hosts: ["your_private_ip:9200"]

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  username: "elastic"
  password: "6kNbsxQGYZ2EQJiqJpgl"

. . .
```

Donde *password* es la contraseña que hemos generado en el apartado de configuración de elasticsearch cuyo usuario es *elastic*

Para habilitar los módulos de Suricata y Zeek ne filebeat debemos ejecutar lo siguiente.

```
sudo filebeat modules enable suricata
sudo filebeat modules enable zeek
```

Por último ejecutaremos la siguiente instrucción para terminar la configuración de filebeat.

```
sudo filebeat setup
```

Si todo ha funcionado correctamente, el comando anterión debería generarnos una salida como la que se ve a continuación.

```
Output
Overwriting ILM policy is disabled. Set `setup.ilm.overwrite: true` for enabling.

Index setup finished.
Loading dashboards (Kibana must be running and reachable)
Loaded dashboards
Setting up ML using setup --machine-learning is going to be removed in 8.0.0. Please use the ML app instead.
See more: https://www.elastic.co/guide/en/machine-learning/current/index.html
It is not possible to load ML jobs into an Elasticsearch 8.0.0 or newer using the Beat.
Loaded machine learning job configurations
Loaded Ingest pipelines
```
