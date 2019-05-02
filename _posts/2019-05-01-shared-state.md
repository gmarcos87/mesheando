---
layout: post
title: "Introducción a shared-state"
subtitle: "Distribuyendo estado como una mesh manda"
date: 2019-05-01 08:45:13 -0400
categories: Tutoriales
author: Marcos
background: "/assets/backgrounds/lime-lua-bg.jpg"
---

## Bienvenido shared-state a la familia LibreMesh
Muches pensarían que la capa de coordinación entre los nodos se termina con el protocolo y que con eso ya es suficiente. Pero no, coordinar las direcciones de IP asignadas en los distintos nodos y compartir los nombres de los hosts en toda la red son dos tareas que exceden a los protocolos de ruteo mesh y que, valiéndose de ellos, se implementaron por separado.
Aunque me conecte al nodo PepitoLibre y luego me vaya a JosefaLibre, quiero mantener mi dirección ip. Como también si tengo una compu con un servidor de películas, quiero que otres integrantes de la red puedan poner https://peliculas.lan y acceder sin tener que conocer el ip de esa computadora (algo como 10.13.128.53).

Para eso (asignación de hosts en dnsmasq) utilizábamos Alfted, el mayordomo de Batman, en dos paquetes llamados [dnsmasq-distributed-hosts](https://github.com/libremesh/lime-packages/tree/master/packages/dnsmasq-distributed-hosts) y [dnsmasq-lease-share](https://github.com/libremesh/lime-packages/tree/master/packages/dnsmasq-lease-share).
Alfred tiene un esquema de distribución master-slave, en donde uno puede emitir o escuchar determinado id y recibir la información que el master coordina y envía. Luego, con esa información se puede hacer lo que se quiera, en este caso reconstruir las tablas de hosts de dnsmasq de toda la red en cada nodo.

El problema es que Alfred falla mucho en nuestra escala de redes, además el esquema master-slave funciona sólo si la red es estable y puede asegurar la conectividad de los slave a los master. En QuintanaLibre no funcionó, el tamaño de la red impidió contar con un solo master, y cuando quisimos agregar más la red colapsó.
A eso le sumamos la incapacidad de poder debuguear correctamente qué es lo que estaba pasando.

Entonces, nos sentamos, discutimos un poco y nos decidimos por diseñar un sistema en el que la información fluya entre los nodos vecinos (a esto se lo llama gossip o chisme) y que, eventualmente, todos los nodos cuenten con la misma información, o algo parecido. El encargado de programarlo fue Gioacchino (Gio) y el resultado se llama shared-state.

## Funcionamiento
Podemos pensar a shared-state como una biblioteca. Tiene distintas estanterías, en donde cada una se encarga de una temática particular. En esas estanterías hay libros, cada uno tiene un código único. A su vez, cada libro posee autor, fecha de publicación y contenido. Espero que hasta acá, la metáfora se entienda. 

![Ubatuba](/assets/posts/shared-state-estanterias.png){:class="center img-fluid"}
<span class="caption text-muted">/var/shared-state/data el hogar de las estanterías</span>

Cuando shared-state quiere sincronizar su biblioteca busca otras bibliotecas a la vista (esto lo hace shared-state-get_candidates_neigh escaneando los clientes/nodos asociados) y le solicita las novedades de determinada estantería. 
La sincronización de las estanterías puede ser producto de algún evento que pase en el nodo o de forma periódica (cron), pero de todas formas el intercambio es mas o menos así: 

- Hola Biblioteca B, yo soy Biblioteca A, estos son mis libros de la estantería Pedagogía, tienen su código único, además de su fecha de publicación, autor y contenido.
- Gracias Biblioteca A, estos son los míos.

Luego de este respetuoso diálogo (via HTTP y como json), shared-state puede comparar y actualizar su biblioteca en algo que se denomina *merge*, que sigue estas dos reglas:
- Si no tengo el libro con ese código único lo agrego a la estantería de mi biblioteca.
- Si ya tengo un libro con ese código único, guardo el que tenga la fecha de publicación mas reciente y tiro el viejo

Además, existen dos elementos más: la publicación y los hooks. La publicación es la forma en que los datos llegan a shared-state, son scripts que le inyectan la información. Por ejemplo, el script de [hosts de dnsmasq](https://github.com/libremesh/lime-packages/tree/master/packages/shared-state-dnsmasq_hosts) analiza los registros del nodo, los transforma en un json y lo envía a shared-state. A cada uno de estos contenidos, shared-state le asigna un código único, fecha de publicación y autor. Esos contenidos ya están listos para circular. Los hooks corren en el sentido inverso, luego de que shared-state actualiza determinada estantería de la biblioteca, va y busca a todos aquellos que quieran escuchar las novedades y se las notifica. En el ejemplo de dnsmasq, lo que sucede es que con el nuevo json reconstruye los archivos de hosts y direcciones IPs asignadas, luego reinicia a dnsmasq para que tome la nueva configuración. En otras palabras: publicación y hooks, entrada y salida de información entre el sistema y shared-state.

## Pirania, y los vouchers distribuidos
[Pirania](https://github.com/libremesh/pirania/) es el sistema de portal cautivo con acceso a internet vía vouchers, que estamos programando en AlterMundi. Decidimos hacer distribuida la tabla de códigos entre todos los nodos y para ello usamos shared-state.

![Ubatuba](/assets/posts/pirania-logo.png){:class="center img-fluid"}
<span class="caption text-muted">Solución de Voucher y Portal Cautivo para redes comunitarias</span>

La particularidad que tiene [shared-state-pirania](https://github.com/libremesh/lime-packages/tree/master/packages/shared-state-pirania) (como plugin para shared-state) es que, no solo necesitamos agregar datos sino también editar los existentes.
Esto no parece un desafio, pero en realidad sí lo es. Volvamos a mirar como hace el merge shared-state, pero esta vez en el contexto de Pirania:

- Si no tengo un voucher con ese código único lo agrego al listado de vouchers
- Si ya tengo ese voucher con ese código único, guardo el que tenga la fecha de publicación más reciente y tiro el viejo.

El problema es que finalmente persiste el voucher que tenga la fecha de publicación mas nueva, que no necesariamente es el que está más actualizado, ya que la fecha de publicación está relacionada al momento en el que el dato es enviado a shared-state, y puede que en otro nodo se publique recientemente un voucher viejo o sin actualizar, y eso termine pisando en otro nodo el voucher actualizado pero no publicado recientemente. Cuando hicimos este intento notamos como los estados se pisaban entre sí, y nunca llegaban a una concurrencia, las ediciones volvían para atrás, una y otra vez. 

La conclusión fue: no es shared-state el que tiene que hacer el merge de los contenidos, y el código no debe referir a un id (código del voucher) sino a un hash del contenido _([ver commit c8285c](https://github.com/libremesh/lime-packages/pull/507/commits/c8285cf41a23105bc99a5e1b3405119225a9db9d))_.

> **¿Qué es un hash?**
>Es el resultado de una función en la que se obtiene una determinada cantidad de caracteres como producto de una misma entrada. Al cambiar algo de la entrada, el resultado es diferente. Podemos decir que, el hash es como la huella digital de algo. Por ejemplo, el hash tipo **md5** de "hola" es "916f4c31aaa35d6b867dae9a7f54270d", mientras que el de "Hola" es "8c8432c5523c8507a5ec3b1ae3ab364f". Completamente diferentes, porque "hola" y "Hola", no son  idénticos.

Si el contenido no cambia, la id en shared-state será la misma y el merge lo hace shared-state según su lógica. Si en cambio, el contenido de un voucher cambió (alguien registró su dirección mac o se invalidó) el hash del contenido también cambia, por ende la id en shared-state será distinta y no pisará los resultados anteriores. Viajarán tanto los estados originales como los alterados.

Luego, el encargado de recibir los datos de shared-state y transformarlo en algo útil, es el hook de pirania. Ahí tenemos [*un segundo merge*](https://github.com/libremesh/lime-packages/pull/507/commits/d786e17b0fd5ddd75230dac0b24a2486208c11c5#diff-12b5b254e16a895dfd35b05f23dc5867R61), que funciona con otra lógica:

- Si solo un contenido hace referencia al voucher, uso ese contenido
- Si hay más de un contenido que hace referencia a un voucher en particular:
	- Uso el que tiene más macs asignadas (alguien agregó su mac al voucher)
	- Uso el que tiene tiempo válido en cero (voucher anulado)

Listo, eventualmente los estados primitivos se dejarán de replicar y solo circularán los modificados.

## Desafíos
Shared-state no propone nada nuevo en sí, existen muchos sistemas como este y se denominan CRDT ([Replica de información libre de conflicto](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)) o también ["Eventual consistencty"](https://en.wikipedia.org/wiki/Eventual_consistency).
Los principales desafíos son documentar y crear una serie de herramientas de desarrollo y debugueo que permita a les desarrolladores sumar y mejorar esta pequeña pero poderosa pieza de código. Creo que puede aportar mucho en las redes mesh e incluso conectar otros dispositivos dentro del esquema.

Tranquilamente se puede utilizar shared-state para anunciar servicios de sincronización mas "pesados" que corran en celulares u otros dispositivos... _alguien dijo mensajería? redes sociales? cryptomonedas? contenidos locales?_  o incluso hacer epidémicos algunos cambios de configuración en la mesh.