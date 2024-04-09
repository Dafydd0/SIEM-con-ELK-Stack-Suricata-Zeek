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
