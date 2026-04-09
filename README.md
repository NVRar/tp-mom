# Trabajo Práctico - Middlewares Orientados a Mensajes

Los middlewares orientados a mensajes (MOMs) son un recurso importante para el control de la complejidad en los sistemas distribuídos, puesto que permiten a las distintas partes del sistema comunicarse abstrayéndose de problemas como los cambios de ubicación, fallos, performance y escalabilidad.

En este repositorio se proveen conjuntos de pruebas para los dos formas más comunes de organización de la comunicación sobre colas, que en RabbitMQ se denominan Work Queues y Exchanges.

Se recomienda familiarizarse con estos conceptos leyendo la documentación de RabbitMQ y siguiendo los [tutoriales introductorios](https://www.rabbitmq.com/tutorials).

## Condiciones de Entrega

El código de este repositorio se agrupa en dos carpetas, una para Python y otra para Golang. Los estudiantes deberán elegir **sólo uno** de estos lenguajes y completar la implementación de las interfaces de middleware provistas con el objetivo de pasar las pruebas asociadas.

Al momento de la evaluación y ejecución de las pruebas se **descartarán** los cambios realizados a todos los archivos, a excepción de:

**Python:** `/python/src/common/middleware/middleware_rabbitmq.py` 

**Golang:** `/golang/internal/factory/*/*.go` 

## Ejecución

`make up` : Inicia contenedores de RabbitMQ  y de pruebas de integración. Comienza a seguir los logs de las pruebas.

`make down`:   Detiene los contenedores de pruebas y destruye los recursos asociados.

`make logs`: Sigue los logs de todos los contenedores en un solo flujo de salida.

`make local`: Ejecuta las pruebas de integración desde el Host, facilitando el desarrollo. Se explica con mayor detalle dentro de su sección.

## Pruebas locales desde el Host

Habiendo iniciado el contenedor de RabbitMQ o configurado una instancia local del mismo pueden ejecutarse las pruebas sin necesidad de detener y reiniciar los contenedores ejecutando `make local`, siempre que se cumplan los siguientes requisitos.

### Python
Instalar una versión de Python superior a `3.14`. Se recomienda emplear un gestor de versiones, como ser `pyenv`.
Instalar los dependencias de la suite de pruebas:
`pip install -r python/src/tests/requirements.txt`

### Golang
Instalar una versión de Golang superior a `1.24`.
Instalar los dependencias de la suite de pruebas:
`go mod download`

## Resolución

### Parte 1: MessageMiddlewareQueueRabbitMQ
Acá la idea es implementar Work Queues. Por eso termino usando `durable=True` para que la queue sobreviva si se cae RabbitMQ. Acompañando esto también va el `delivery_mode=2` cuando se publica un mensaje para asegurar de que se guarde en disco por si se tiene que reenviar frente a una caída, el mensaje está persistido. 

No declaro un exchange explícito, sino que uso el default por eso le paso ''
Uso prefetch_count=1 para obligar a RabbitMQ a que no le entregue un nuevo mensaje al consumidor hasta recibir su ACK. Con esto evito que un solo consumidor rápido se lleve todos los mensajes de la cola.

### Parte 2: MessageMiddlewareExchangeRabbitMQ
Acá sí defino un exchange explícito con el nombre recibido por parámetro. Además, a ese exchange lo hago de tipo direct así matchea de forma estricta con las routing keys dadas. Un exchange tipo fanout no serviría porque le mandaría a todos lo mismo ignorando la ruta, y no es el propósito de esto. Un topic podría ser, pero me parece más para cuando en las reglas usas # y *, y entiendo que para este ejercicio lo que mejor se adapta es direct.

Acá sería un Publish/Subscribe. Así que al instanciarse la clase, tanto para usarse como publisher o como consumer, se declara el exchange que es una acción idempotente.

Cada consumidor necesita no compartir la queue, así que por eso va `exclusive=True`. Además, es una queue que no necesita un nombre específico así que se lo dejo a RabbitMQ pasando un string vacío ''. La queue de cada consumer se borra al desconectarse. 

Acá no uso durable=True ni tampoco `delivery_mode=2`. La idea es que el producer publica algo y solo lo escuchan los consumidores que están contectados en ese momento. Si el consumer se desconecta, no tiene sentido persistir en disco un mensaje para una queue temporal que se borró al salir el consumer. 

Es importante el for al momento del bindeo porque un mismo consumer puede necesitar estar suscrito a múltiples routing keys a la vez usando su misma queue.

El método `send()` no permite aclarar una routing key por parámetro en el momento de envío. Por lo que dentro de este hago un `for` y publico el mensaje al exchange por cada una de las routing keys guardadas al inicializar.

Por como está ahora si un consumer está suscripto por ejemplo a route_1 y también a route_2, y el producer manda el mensaje "Hola" usando send, entonces al consumer le van a llegar dos mensajes "Hola" repetidos, uno por cada routing key que el producer mandó.
