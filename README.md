# Kafka Connect 101

## Introducción

https://docs.confluent.io/platform/current/connect/concepts.html

https://docs.confluent.io/3.1.1/_images/converter-basics.png

```
Alguna fuente ->

    Kafka Connect Worker

        Source Connector

            Source Task (lee de fuente, crea un ConnectRecord) -> Single Message Transformations (optional) -> Converter (serializa ConnectRecord a mensaje Kafka)

-> Tópico en Kafka

Tópico en Kafka ->

    Kafka Connect Worker

        Sink Connector

            Converter (deserializa mensaje Kafka a ConnectRecord) -> Single Message Transformations (optional) -> Sink Task (escribe en destino)

-> Algún destino
```

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

## Errores

### Troubleshooting

https://developer.confluent.io/learn-kafka/kafka-connect/troubleshooting-kafka-connect/

* Conectar por API Rest para ver último error y estado

### Tratamiento de errores y dead-letter-queue

https://developer.confluent.io/learn-kafka/kafka-connect/error-handling-and-dead-letter-queues/

* Error tolerances, lo que hace y como configurarlo
* Dead letter queue, estrategias, por ejemplo simplemente configurar otra tarea que lee de la DLQ

## Crear un connector sink desde 0

https://github.com/apache/kafka/tree/2.8/connect/file/src/main/java/org/apache/kafka/connect/file

* FileStreamSinkConnector simplemente, según número configurado de tasks, devuelve la configuración de cada Task
* FileStreamSinkTask se instancia n veces según número configurado de tasks, y en método put recibe los Records y los escribe en el outputStream