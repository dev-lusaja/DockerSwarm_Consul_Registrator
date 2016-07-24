
DOCKER SWARM + Consul + Registry
================================

La arquitectura tendra el siguiente esquema:

* 1 Servidor para Service Discover (Consul)
* 1 Servidor Swarm Master
* 2 Servidores Nodos para Swarm master con un registrador de servicios.

Requisitos:
-----------

Crear 4 maquinas virtuales con el driver "VB"

Servidor Consul
---------------
Crear un archivo docker-compose.yml
~~~
myconsul:
  image: progrium/consul
  restart: always
  hostname: consul
  ports:
    - 8400:8400  
    - 8500:8500
  command: "-server -bootstrap"
~~~

Creamos la maquina virtual
~~~
$ docker-machine create -d=virtualbox consul-machine
~~~

Configuramos el cliente de docker hacia la maquina virtual de consul
~~~
$ eval $(docker-machine env consul-machine)
~~~

Creamos e iniciamos en modo demonio el contenedor de consul
~~~
$ docker-compose up -d
~~~

Servidor Swarm Master
---------------------

Creamos el swarm master utilizando el service discover de consul
~~~
$ docker-machine create -d virtualbox --swarm --swarm-master --swarm-discovery="consul://$(docker-machine ip consul-machine):8500" --engine-opt="cluster-store=consul://$(docker-machine ip consul-machine):8500" --engine-opt="cluster-advertise=eth1:2376" swarm-master
~~~

> --swarm = aprovisiona la maquina con la imagen de swarm.

> --swarm-master = configura la maquina como la principal del cluster.

> --swarm-discovery = indica la dirección del servidor discovery.

> --cluster-store = indica donde se encuentran el almacenamiento de los token.

> --cluster-advertise = indica en que red se anunciara la maquina

Configuramos el cliente de docker hacia la maquina virtual del Swarm Master
~~~
$ eval $(docker-machine env swarm-master)
~~~

Verificamos la información del servidor creado
~~~
$ docker info
~~~

Servidor Nodo(s)
----------------

Crearemos un nodo que servira como ejemplo para la creación de muchos más
~~~
$ docker-machine create -d virtualbox --swarm --swarm-discovery="consul://$(docker-machine ip consul-machine):8500" --engine-opt="cluster-store=consul://$(docker-machine ip consul-machine):8500" --engine-opt="cluster-advertise=eth1:2376" swarm-node-01
~~~

* Si se desea crear mas nodos se puede utilizar este comando solamente cambiando el nombre de la maquina.

Configuramos el cliente de docker hacia la maquina virtual del nodo
~~~
$ eval $(docker-machine env swarm-node-01)
~~~ 

En cada agente que creado debemos colocar un registrador para que este escuchando todos los cambios de estado en los contenedores y estos sean registrados en Consul
~~~
$ docker run -d \
    --name=registrator \
    --net=host \
    --volume=/var/run/docker.sock:/tmp/docker.sock \
    gliderlabs/registrator:latest \
      consul://$(docker-machine ip consul-machine):8500
~~~

Verificación de Cluster
-----------------------

Configuramos el cliente de docker hacia la maquina virtual del Swarm Master
~~~
$ eval $(docker-machine env --swarm swarm-master)
~~~
* Observe que le estamos indicando que deseamos entrar a ver la información del cluster de Swarm.

Obtenemos información de Swarm y los nodos que estan agregados
~~~
$ docker info
~~~

Podemos tambien tener información de los servidores que estan en el cluster a traves de consul
~~~
$ docker run swarm list consul://$(docker-machine ip consul-machine):8500
~~~

Consideraciones
---------------

* Si trabajamos con docker-compose debemos saber que no podemos utilizar el flag "volumes_from" porque cada contenedor se puede crear en distinto nodo, lo cual es una restricción que tiene swarm.
* Las aplicaciones creadas con docker-compose debe tener labels o environments con el prefijo "SERVICE_XXX" todo esto servira como metadata para que se registre en consul.
 
Referencias
-----------

~~~
https://dzone.com/articles/docker-swarm-cluster-using-consul
http://gliderlabs.com/registrator/latest/user/services/
~~~