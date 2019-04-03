---
layout: post
title: "OpenWrt, Libremesh y Lua"
subtitle: "...por algún lado hay que comenzar"
date: 2019-03-13 23:45:13 -0400
categories: Tutoriales
author: Marcos
background: "/assets/backgrounds/lime-lua-bg.jpg"
---

## En el principio...

Se me ocurrieron mil ideas (tal vez un poco menos) sobre el primer post de este blog. Luego me di cuenta que el primero realmente no es importante, posiblemente sea el menos leido y quede olvidado en el fondo del cajón. Ya lo dijo Yisus, en el principio fue la palabra o en nuestro caso el lenguaje, y el lenguaje en OpenWrt y LibreMesh es Lua (luna en portugués).

En realidad son varios los lenguajes utilizados, vas a encontrar scripts en bash, muchas utilidades y protocolos escritos en C, e incuso lime-app en JavaScript. Pero los scripts de LibreMesh, y gran parte de todo aquello que este blog pretende cubrir, está más relacionado pequeños scripts en Lua que con programar drivers en C o cambios en el kernel de linux.

## ¿Que es Lua?

Lua es un lenguaje de programación interpretado, es decir que uno escribe instrucciones (digamos el programa) y que para que corra ese script (archivo de texto) pasará por un interprete que lo transforma en una serie de instrucciones que la máquina (router en nuestro caso) entiende.
En otros casos, como el lenguaje C, la interpretación se realiza una sola vez en un proceso que se llama compilación, y el resultado es un programa que corre directamente y que no necesita ser interpretado.

La gran diferencia es que al script de Lua lo podemos ir editando y corriendo constantemente mientras que a un programa escrito en C es necesario recompilarlo si le hacemos algún cambio.

Es una relación costo/beneficio, interpretar constantemente un script es un proceso extra y el rendimiento del resultado suele ser inferior al de un programa en C compilado. Pero la posibilidad de abrir un script en un editor de texto y cambiar su funcionamiento es algo que a mi entender ayuda a desarrollar y aprender mas rápido.

## ¿Porqué Lua y no Python, JavaScript, ... ?

OpenWrt ya incluye en su diseño el interprete de Lua, si bien uno podría compilar el interprete de Python o incluso node.js para que corra en OpenWrt nos encontramos con otra variable a considerar, la capacidad de almacenamiento de los dispositivos. Estamos hablando de 4mb a 8mb en la mayoría de los casos para todo: sistema operativo, protocolos, scripts, webui, etc. Entonces acoplarse a las decisiones de arriba (upstream a partir de ahora) es una forma de ahorrar espacio. Ademas nos mantiene en linea con la comunidad ya que podemos hablar en un mismo lenguaje (aunque en un idioma distinto… grrrrrr – aprende ingles de una vez!)

## Mi recorrido personal

Mi lenguaje favorito es JavaScript, es en el que más cómodo me siento, no es le mejor, en un lenguaje. Lo importante es como se utiliza, he visto código JS escrito de mil formas, programadores de C escriben código JS muuy similar a C y programadores de Java meten classes por todos lados. Lo que si es simpático es ver a programadores de Python arder en un infierno de callbacks jejeje ← chiste nerd :)

Lo primer que hice fue buscar un transpiler, que es como un compilador pero que en lugar de pasar el código a instrucciones de máquina para el código de un lenguaje a otro. No lo recomiendo, lo mejor es aprender el lenguaje, mirar el código ya escrito en LibreMesh y tratar de comprender que es lo que se esta haciendo, de a poco pasaras de hacer copia y pega a escribir tus propias funciones y loops.
