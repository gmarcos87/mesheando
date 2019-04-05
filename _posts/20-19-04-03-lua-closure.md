---
layout: post
title: "Closures, filter, map y reduce en Lua"
subtitle: "Caso de uso real en LibreMesh"
date: 2019-03-04 08:45:13 -0400
categories: Tutoriales
author: Marcos
background: "/assets/backgrounds/lime-lua-bg.jpg"
---

## Qué es una clousure?
>Un closure o clausura es la combinación de una función y el ámbito léxico en el que se declaró dicha función. Es decir los closures o clausuras son funciones que manejan variables independientes. En otras palabras, la función definida en el closure "recuerda" el ámbito en el que se ha creado.

Esa es la definición que se encuentra en [Mozilla Docs](https://developer.mozilla.org/es/docs/Web/JavaScript/Closures), resumiendo: Una función que retorna otra función, que a su vez mantiene el estado de la primera.

En este ejemplo hago una función sumadora, que retorna una función que suma siempre el monto con el que se creó: 

{% highlight lua %}
local function sumador(cantidad)
    return function(monto)
        return cantidad + monto
    end
end

local agregar_cuatro = sumador(4)
local agregar_cinco = sumador(5)

print(agregar_cautro(8)) -- 12
print(agregar_cinco(16)) -- 21

{% endhighlight %}


## Caso de uso

Actualmente estoy colaborando con el desarrollo del [First Boot Wizard de LibreMesh](https://github.com/libremesh/lime-packages/tree/master/packages/first-boot-wizard), un asistente que permite analizar las redes cercanas, elegir una y configurarse como parte de esa red. En caso de que no encuentre ninguna puede crear una de forma rápida y simple.

El objetivo es poder facilitar al máximo la creación y expansión de redes comunitarias. Agregar un nodo es prender, hacer tres clicks y listo, a disfrutar de la mesh.

Para que esto ocurra el proceso en el router es el siguiente:

- Chequear si el nodo esta ya configurado o esta "nuevo"
- Si está configurado, ignora el escaneo y continúa normalmente
- Si no está configurado, realiza el escaneo de las redes

Luego del escaneo y el filtrado (sólo necesitamos las redes mesh y adhoc) es necesario realizar un ordenamiento según tipo de redes y de canales. De esta forma, evitamos cambiar múltiples veces la configuración de la interfaz de red para conectarnos y obtener la data de autoconfiguración. El cambio de canal y modo es un cambio costoso en términos de tiempo, la red puede llegar a estar sin respuesta por varios segundos. Si el nodo ve a otros 5 nodos mesh, y reconfigura la interfaz cada vez que intenta conectarse, puede tardar hasta un minuto y medio en obtener las configuraciones que de forma optimizada puede realizar en 10 segundos. Es, en este ordenamiento, en donde vamos a usar las clousures. Pero, por ahora, mejor meternos un poco en el código previo, la obtención de las redes.

## Escaneando las redes

El código encargado de obtener las redes es el siguiente:

{% highlight lua %}
local iwinfo = require("iwinfo")
local wireless = require("lime.wireless")

local function phy_to_idx(phy)
    local substr = string.gsub(phy, "phy", "")
    return tonumber(substr)
end

function get_networks()
    local phys = {}
    local all_networks = {}
    local all_radios = wireless.scandevices()

    -- Get physical interfaces 
    for *, radio in pairs (all_radios) do
        if wireless.is5Ghz(radio[".name"]) then
            local phyIndex = string.sub(radio[".name"], -1)
            phys[#phys+1] = "phy"..phyIndex
        end
    end

    -- Scan networs
    for idx, phy in pairs(phys) do
        networks = iwinfo.nl80211.scanlist(phy)
        for k,network in pairs(networks) do
            if network.signal ~= -256 then
                network["phy"] = phy
                network["phy_idx"] = phy_to_idx(phy)
                all_networks[#all_networks+1] = network
            end
        end
    end

    return all_networks

end
{% endhighlight %}

Vamos a analizar parte por parte este código. Al principio, se declaran dos variables como tabla. Servirán para almacenar las radios que tiene el router y el listado de redes. Además, mediante una llamada a la librería wireless de LibreMesh obtenemos todos los dispositivos WiFi del router.

{% highlight lua %}
    local phys = {}
    local all_networks = {}
    local all_radios = wireless.scandevices()
{% endhighlight %}

*all_radios* tendrá una estructura similar a esta (puede variar según el modelo de router)

```json
{
  "radio0": {
    ".name": "radio0",
    "type": "mac80211",
    "disabled": "0",
    "country": "US",
    "distance": "100",
    "noscan": "1",
    "hwmode": "11g",
    ".index": 0,
    "legacy_rates": "1",
    "channel": "3",
    ".type": "wifi-device",
    ".anonymous": false,
    "htmode": "HT20",
    "path": "platform/ar934x_wmac"
  },
  "radio1": {
    ".name": "radio1",
    ".anonymous": false,
    "disabled": "0",
    "distance": "1000",
    "hwmode": "11a",
    ".index": 1,
    "path": "pci0000:00/0000:00:00.0",
    "channel": "48",
    "noscan": "1",
    ".type": "wifi-device",
    "htmode": "HT40",
    "type": "mac80211"
  }
}
```

Para realizar el scan es necesario obtener el nombre de las interfaces físicas, generalmente es pyh1 para radio0, phy1 para radio1, etc; pero puede variar según el dispositivo, por eso mejor los obtenemos consultando al sistema. 

{% highlight lua %}
    for *, radio in pairs (all_radios) do
        if wireless.is5Ghz(radio[".name"]) then
            local phyIndex = string.sub(radio[".name"], -1)
            phys[#phys+1] = "phy"..phyIndex
        end
    end
{% endhighlight %}

En este loop revisamos que la radio sea de 5ghz, ya que si escaneamos en 2.4ghz desconectaríamos a los clientes, incluido al que está tratando de configurar el router (si es por WiFi). Si es de 5ghz, utilizamos el último caractér para armar el nombre de la interfaz.

Teniendo este listado podemos hacer el escaneo usando la librería *libiwinfo* que expone, en Lua, información similar al comando *iw* en la terminal.

{% highlight lua %}
    for idx, phy in pairs(phys) do
        networks = iwinfo.nl80211.scanlist(phy)
        for k,network in pairs(networks) do
            if network.signal ~= -256 then
                network["phy"] = phy
                network["phy_idx"] = phy_to_idx(phy)
                all_networks[#all_networks+1] = network
            end
        end
    end
{% endhighlight %}

Un detalle a considerar es que si la red está siendo emitida por la propia interfaz, el nivel de señal es de -256, por lo que podemos filtrar esas redes tomando sólo las que son distintas (~=) a ese nivel.

La tabla networks que retorna *iwinfo.nl80211.scanlist* va a variar según las redes que el dispositivo esté viendo, pero para que tengas una idea de la estructura éste es el resultado de un scan real:

```json
[{
  "encryption": {
    "enabled": false,
    "auth_algs": [],
    "description": "None",
    "wep": false,
    "auth_suites": [],
    "wpa": 0,
    "pair_ciphers": [],
    "group_ciphers": []
  },
  "quality_max": 70,
  "ssid": "LiMe",
  "channel": 48,
  "signal": -38,
  "bssid": "F8:1A:67:33:0B:68",
  "mode": "Mesh Point",
  "quality": 70
}, {
  "encryption": {
    "enabled": false,
    "auth_algs": [],
    "description": "None",
    "wep": false,
    "auth_suites": [],
    "wpa": 0,
    "pair_ciphers": [],
    "group_ciphers": []
  },
  "quality_max": 70,
  "ssid": "ubatuba.libre",
  "channel": 48,
  "signal": -40,
  "bssid": "FA:1A:67:33:0B:68",
  "mode": "Master",
  "quality": 70
}, {
  "encryption": {
    "enabled": false,
    "auth_algs": [],
    "description": "None",
    "wep": false,
    "auth_suites": [],
    "wpa": 0,
    "pair_ciphers": [],
    "group_ciphers": []
  },
  "quality_max": 70,
  "ssid": "ubatuba.libre/LiMe-gateway",
  "channel": 48,
  "signal": -39,
  "bssid": "FE:1A:67:33:0B:68",
  "mode": "Master",
  "quality": 70
}]
```
Lo que hace el loop *for* es agregar, a cada una de esas redes, dos campos: **phy** y **phy_index** que serán necesarios para el resto del proceso. **phy** es el nombre de la interfaz en la que fue capturada esa red y **phy_index** el valor numérico de esa interfaz. En este caso **"phy1"** y **1** respectivamente. Luego, las va insertando a la tabla all_networks que finalmente es retornada.

## Functols
El código de First Boot Wizard utiliza una serie de utilidades de programación funcional: filter, map y curry. Está última en menor medida. 

### ft.map
*ft.map* recibe dos valores, una tabla y una función. Lo que hace, es aplicar la función a cada uno de los elementos de la tabla y retornar su resultado.
Volvamos al ejemplo del sumador del comienzo:

{% highlight lua %}
ft = require('firstbootwizard.functools')
json = require("luci.json")

-- Creamos la función sumadora (Closure)
suma_dos = sumador(2)

-- Creamos una tabla con algunos valores
valores = {}
for i = 10,1,-1 do 
    valores[#valores+1] = i
end
-- Usamos Map para aplicar la función sumadores a cada elemento de **valores**
valores_nuevos = ft.map(suma_dos, valores)

print('Valores: '..json.encode(valores))
-- Nuevos valores [10,9,8,7,6,5,4,3,2,1]

print('Nuevos valores: '..json.encode(valores_nuevos))
-- Nuevos valores [12,11,10,9,8,7,6,5,4,3]

{% endhighlight %}

## ft.filter
Es como Map, en el sentido de que itera sobre una tabla y aplica una función a cada uno de los elementos, la diferencia es que la función debe retornar true o false, si es true el elemento es retornado y si es false es filtrado.

Por ejemplo, la función de filtrado de las redes es bastante simple, solo nos interesan las redes mesh y adhoc, el resto las descartamos.

{% highlight lua %}
local ft = require('firstbootwizard.functools')
local fbw = require('firstbootwizard')

local all_networks = get_networks()

local function filter_mesh(n)
    return n.mode == "Ad-Hoc" or n.mode == "Mesh Point"
end

local only_mesh = ft.filter(filter_mesh, all_networks)
{% endhighlight %}

## Finalmente: ordenamiento
Retomemos el listado de redes obtenidas por get_networks y procesadas por el filtro *filter_mesh*. Los únicos campos que nos interesan son el canal (channel) y el tipo de red (mode). Vamos a escribir un mockup del tipo de datos que necesitamos procesar:

{% highlight lua %}
local json = require('luci.json')
local mockaup_networks = json.decode([[
 [
  {
    "channel": 161,
    "mode": "adhoc"
  },
  {
    "channel":  48,
    "mode": "adhoc"
  },
  {
    "channel": 161,
    "mode": "mesh"
  },
  {
    "channel": 161,
    "mode": "adhoc"
  },
  {
    "channel": 36,
    "mode": "mesh"
   }
]
]])
{% endhighlight %}

Son un puñados de redes, algunas mesh y otras ad-hoc, en distintos canales. Primero, separemos las redes según su modo:

La opción mas tradicional sería hacer un for y agregar la red a una u otra tabla previamente seleccionada:
{% highlight lua %}
 result = {}
 for k, network in pairs(mockaup_networks) do
      if result[obj.mode == nil then
        result[obj.mode] = {}
      end
      table.insert(result[obj.mode], obj)
    end
    return result
{% endhighlight %}

El problema con esto, es que si luego queremos separar otra tabla por otro tipo de campo debemos reescribir la función, y como se trata de ahorrar tiempo de desarrollo y especialmente, espacio en la memoria del router, es mejor separar, reciclar y reutilizar (como la basura pero con código).

Convirtamos ésto en una función clousure, en donde pasemos el parámetro que queremos utilizar como separador y obtengamos una función separadora.

Lo primero será armar el esqueleto, básicamente el clousure es una función que retorna otra función, así que:

{% highlight lua %}
function splitBy(option)
    return function(tab)
    end
end
{% endhighlight %}

Ahora adaptemos el *for* a este nuevo esquema

{% highlight lua %}
function splitBy(option)
  return function(tab)
    local result = {}
    for k, obj in pairs(tab) do
      if result[obj[option]] == nil then
        result[obj[option]] = {}
      end
      table.insert(result[obj.mode], obj)
    end
    return result
  end
end
{% endhighlight %}

**Listo!** cuando llamemos a la función retornará otra configurada según la *option* que le pasemos.

{% highlight lua %}
local splitByMode = splitBy('mode')
local networks = splitByMode(mockaup_networks)

-- Incluso se puede llamar en línea
local networks_by_channel = splitBy('channels')(mockup_networks)
{% endhighlight %}

El resultado es una tabla de tablas
```json
{
    "mesh": {...},
    "ad-hoc": {...}
}
```
Deberíamos llamar una función de ordenamiento en cada una de estas subtablas y guardar el resultado: eso suena mucho a lo que hace ft.map, no? Escribamos primero como quedaría el código y luego lo rellenamos con la lógica.
 
{% highlight lua %}
data = splitBy('mode')(mockup_data)
data = map(shortBy('channel'), data)
{% endhighlight %}

Lua trae la utilidad **sort**, a la que debes pasarle una función que recibe dos parámetros, left y right, mediante una comparación puedes determinar si el orden es mayor o menor. El resultado, es a algo así como una tabla ordenada. Digo "algo así" porque orden y tablas son dos palabras difíciles de juntar en Lua. Puedes buscar más al respecto y ver como funcionan los índices y tablas, pero por ahora esto cumple con el objetivo.
Este clousure, al igual que splitBy, se inicia con un valor (option) que será el campo utilizado en la comparación. En nuestro ejemplo sería channel.

{% highlight lua %}
local function shortBy(option)
  return function(tab)
    table.sort(tab, function (left, right)
      return left[option] < right[option]
    end)
    return tab
  end
end
{% endhighlight %}

Lo bueno, es que ahora tenemos una utilidad nueva, que puede ser separada de nuestro caso de uso y reutilizada en cualquier otra parte de LibreMesh.

Pero el resultado todavía no es el que queríamos, seguimos teniendo una tabla de tablas de redes en lugar de una tabla de redes a secas. Debemos reducir el resultado, y para eso vamos a usar un **reducer**

## Reduce, tu amigo fiel
Lo que hace una función **reduce**, es aplicar una función a cada elemento de la tabla al mismo tiempo que mantienen un acumulador con el resultado previo o de incio. Al finalizar este acumulador es retornado. Como las functools no tienen implementado reduce vamos a hacerlo en estas pocas líneas:

{% highlight lua %}
local function reduce(cb, tab, default)
    -- El resultado es igual al valor default en el inicio
    local result = default
    -- Aplico la función (cb) a cada elemento de la tabla (tab)
    for k, act in pairs(tab) do
        -- El resultado de la función es el nuevo result
        -- que será pasado al próximo elemento del loop
        result = cb(result,act) 
    end
    -- Finalmente retorno el acumulador
    return result
end
{% endhighlight %}

Veamos un ejemplo simple, retomemos la tabla con los números del 10 al 1 del comienzo y apliquemos un reduce que concatene los números y retorne un string.

{% highlight lua %}
function concatenar(prev,act)
    return prev..act
end

local concatenados = reduce(concatenar, valores, '')
print(concatenados) --10987654321
{% endhighlight %}

Si en lugar de concatenar caracteres, lo que quisieramos hacer es concatenar tablas (que es nuestro caso) la función sería algo así:

{% highlight lua %}
function flatTable(prev, act) 
  for i=1,#act do
      prev[#prev+1] = act[i]
  end
  return prev
end
{% endhighlight %}

Sumamos esto a lo que tenemos y la ejecución del ordenamiento está lista:

{% highlight lua %}
-- Dividimos por modo de red
data = splitBy('mode')(mockup_data)
-- Ordenamos por canal en cada modo
data = map(shortBy('channel'), data)
-- Juntamos las redes de todos los nodos de forma ordenada
data = reduce(flatTable,data, {})
{% endhighlight %}

En el camino creamos las utilidades: 
- splitBy
- shortBy
- reduce
- flatTable

Todas completamente reutilizables, testeables y aisladas de nuestro caso de uso particular. Nada mal, no?

## Queda trabajo por hacer
En la versión en desarrollo se agregan dos cuestiones más, el ordenamiento debe tener en cuenta el canal y el modo en el que está el dispositivo (para no iniciar con un cambio que luego hay que volver a aplicar).
Además, si observas la función splitBy el loop *for* puede ser reemplazado por un reduce, ese es un [**buen primer pull-request en LibreMesh**](https://github.com/libremesh/lime-packages/issues?q=is%3Aopen+is%3Aissue+label%3A%22good+first+issue%22), lo estaré esperando.

