# Kafka Connect 101

## Introducción

https://docs.confluent.io/platform/current/connect/concepts.html

## Preparación

```
docker-compose up -d
docker-compose logs -f --tail=0
```

## Interactuar con broker

```
docker-compose exec broker bash
kafka-topics --bootstrap-server localhost:9092 --list
```

## Listar plugins instalados

```
curl -s localhost:8083/connector-plugins | jq
```

## Crear primer connector

Creamos un connector que observa una carpeta de entrada, crea un mensaje por cada fichero que descubre, y mueve ficheros procesados a otra carpeta

```
curl -X PUT -H "Content-Type:application/json" http://localhost:8083/connectors/source-file-01/config -d @connect-file-example/connector-source-config.json
```

Observamos el tópico en otra ventana

```
kafka-console-consumer --bootstrap-server localhost:9092 --topic spooldir-testing-topic --from-beginning --property print.key=true --property print.headers=true --property print.timestamp=true
```

Dejamos ficheros fichero:

```
cp connect-file-example/examples/message1.json connect-file-example/datos/entrada/
cp connect-file-example/examples/message2.json connect-file-example/datos/entrada/
```

## Eliminar connector

```
curl -X DELETE -H "Content-Type:application/json" http://localhost:8083/connectors/source-file-01
```

## Añadir una transformación al conector

Cada tarea en Kafka Connect puede realizar transformaciones sencillas sobre los mensajes entrantes o salientes. Estas transformaciones se llaman SMT (Single Message Transformation), y el nombre ya indica que solo trabajan sobre 1 mensaje a la vez (es decir, no pueden usar datos de otros mensajes, por ejemplo sumar, para ello se usaría Kafka Streams).

Añadimos una que elimina un campo con "información sensible" para que no entre en Kafka: https://docs.confluent.io/platform/current/connect/transforms/maskfield.html

```
curl -X PUT -H "Content-Type:application/json" http://localhost:8083/connectors/source-file-01/config -d @connect-file-example/connector-source-config-with-transformation.json
```

## Serialización hacía Kafka

No configuramos ninguna serialización, así que la tarea usa la serialización configurada por defecto como variable de entorno en el docker-compose.yaml:

```
CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
```

es decir, KEY como string y VALUE en formato Avro.