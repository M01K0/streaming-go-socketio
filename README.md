# Streaming con GO
Streaming en GO un pequeño proyecto, utilizando Socketio.

Buenas chicos, este pequeño proyecto lo lleve a cabo para practicar un poco con GO, el código como veran a continuacion es muy sencillo, fue un proyecto que vi hace un tiempo por internet que hicieron con Nodejs y Socketio y me llamó la atención, ya que no es el típico ejemplo de Chat con socket.

Asi que veran un simple funcionamiento de la Librería de Socketio para Golang, he trabajado en varios proyectos y me ha tocado implementar soluciones en tiempo real, dichas soluciones todas las lleve a cabo con Socketio para Nodejs pero esta vez quise probar la que encontré para Go para este ejemplo, no se si dicha librería abarca hoy día toda la solucion que ofrece la Oficial para NodeJS, pero para la simplicidad de este ejemplo logró encajar perfecta.

### El proyecto

El ejemplo o proyecto es una simple transmisión de imagen entre 1 emisor y múltiples consumidores, es decir un usuario puede iniciarse como emisor a este se le entrega una URL la cual podrá compartir para que los demás puedan consumir esa transmisión y cuenta con un chat grupal entre las personas que consumen y el que emite dicha transmisión.

Varios usuarios pueden iniciarse como emisor y tendrá su url "UNICA" para que los puedan consumir.
<img src="https://programadores.io/wp-content/uploads/2016/09/Captura-de-pantalla-2016-09-22-a-las-1.26.02-p.m.-1024x640.png" alt="captura-de-pantalla-2016-09-22-a-las-1-26-02-p-m" width="1024" height="640" class="alignnone size-large wp-image-201" />

### Comenzamos


Para este articulo necesitamos tener instalado Go:
[Instalando Go](https://programadores.io/instalando-go/)
Utilizaremos la libreria Socketio en Go:
[https://github.com/googollee/go-socket.io](https://github.com/googollee/go-socket.io)

Más las librerías estándar de Go:
* log
* net/http
* strconv

El código es muy simple y la mayoría tiene comentarios que explican su funcionamiento.
Lo primero que debemos hacer es situarnos en nuestro Workspace y en la carpeta de nuestro usuario de github crear un nuevo directorio, en mi caso lo llamare streaming

``mkdir $GOPATH/src/github.com/douglasmakey/streaming``

Luego instalaremos las librerías extras necesarias para este proyecto, como mencione anteriormente solo utilizaremos la librería de Socketio en Go, de resto serán librerías estándar del lenguaje.

En mi caso utilice un administrador de paquetes de Go llamado Glide, si desean saber más sobre esta increíble herramienta tenemos un articulo del mismo [Glide Administrador de paquetes de Go](https://programadores.io/administrador-de-paquetes-en-go/)

También lo podemos hacer con el comando ``go get URL`` que en este caso sería:
``go get github.com/googollee/go-socket.io``, lo que instalara la libreria go-socketio.io,

Luego debemos crear los certificados y la llave privada para poder correr el server con HTTPS, utilizamos los siguientes comandos
``openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout key.pem -out cert.pem``


Crearemos una carpeta 'helpers' en la raiz de nuestro proyecto ``mkdir helpers``, donde guardaremos codigo de ayuda, dentro de esta crearemos el archivo helpers.go, donde delcararemos una funcion para obtener la ip local de la maquina donde ejecutamos el proyecto.

el package de este archivo sera el nombre de la misma carpeta donde se aloja es decir 'helpers' de esta manera podemos seperar logica en go, agrupar o desagrupar funcionalidades dentro de nuestros proyectos.

## Archivo helpers.go
```
package helpers

import "net"

func GetLocalIP() string {
	addrs, err := net.InterfaceAddrs()
	if err != nil {
		return ""
	}
	for _, address := range addrs {
		//Verificamos el tipo de la dirección y si no es una dirección loopback
		if ipnet, ok := address.(*net.IPNet); ok && !ipnet.IP.IsLoopback() {
			//Validamos que sea IPv4 y retornamos la dirección como un string
			if ipnet.IP.To4() != nil {
				return ipnet.IP.String()
			}
		}
	}
	return ""
}
```

 Crearemos el archivo main.go que contendrá el siguiente código:

## Archivo main.go
```go
package main

import (
	"github.com/googollee/go-socket.io"
	"log"
	"net/http"
	"strconv"
)

//Declaramos un tipo transmitter que tendrá la estructura del emisor.
type transmitter struct {
	Id		string				//Id del socket
	so		socketio.Socket			//Socket
}

//Declaramos un tipo consumer que tendrá la estructura del consumidor
type consumer struct {
	Id		string		//Id del socket
	name		string		//Nombre del consumidor
}

//Creamos el tipo namespace que tendrá la estructura del mismo.
type namespace struct {
	name			string				//Nombre del namespace
	counter			int				//Consumidores conectados
	emitter 		*transmitter			//El emisor de dicho Namespace
	consumers 		map[string]*consumer		//Map para que recibe un puntero de la estructura consumidor, para almacenar los consumidores
}
//Creamos un map que recibe un puntero de la estructura namespace para guardar los mismos
var namespaces = make(map[string]*namespace)

//Declaramos la url base del proyecto
var urlBase string = helpers.GetLocalIP()

func main() {
	//Iniciamos el socket
	server, err := socketio.NewServer(nil)
	if err != nil {
		log.Fatal(err)
	}

	server.On("connection", func(so socketio.Socket) {
		log.Println("On connection")

		//Declaramos la variable donde almacenaremos un puntero al Namespace
		var nsp *namespace

		//Capturamos la variable 'type' que envían al conectarse al socket por QueryParam
		tp := so.Request().FormValue("type")
		//Capturamos al nombre del namespace del emisor
		name := so.Request().FormValue("namespace")
		//Seteamos por defecto si no vienen variables.
		if tp == "" {
			tp = "consumer"
		}
		if name == ""  {
			name = "default"
		}

		//Verificamos el tipo
		if tp == "consumer" {
			//Obtenemos el namespace de acuerdo al nombre
			nsp = namespaces[name]
			//Validamos que existe el Namespace, si no desconectamos el socket actual
			if nsp == nil {
				log.Println("Namespace no encontrado")
				so.Emit("disconnect", "Desconectado")
				return
			}

			log.Println("Se ha conectado un nuevo Consumidor al Namespace: " + nsp.name)
			//Capturamos al nombre del usuario consumidor
			user := so.Request().FormValue("user")
			//Si el consumidor no envió su nombre, le asignamos uno.
			if user == "" {
				user = "Consumidor" + strconv.Itoa(nsp.counter)
			}

			//Agregamos el consumidor al MAP de consumidores en el Namespace.
			nsp.consumers[so.Id()] = &consumer{so.Id(), user}

			//Ingresamos a la sala correspondiente al namespace
			so.Join("stream-" + nsp.name)

			//Llevamos el conteo de cuantos consumidores conectados y notificamos al que emite.
			nsp.counter += 1

			//Validamos que emit no este vacio y notificamos al emisor la cantidad de consumidores.
			if nsp.emitter != nil {
				nsp.emitter.so.Emit("count-consume", nsp.counter)
			}
		} else {
			//Type Emisor

			//Creamos el namespace
			nsp = &namespace{name, 0, &transmitter{so.Id(), so}, make(map[string]*consumer)}

			//Guardamos el Namespace en el map de Namespaces
			namespaces[nsp.name] = nsp

			//Log
			log.Println("Se ha conectado un emisor y se crea el namespace: " + name)

			//Ingresamos a la sala correspondiente al namespace
			so.Join("stream-" + name)

			//Definimos la urlBase para los consumidores
			url := "https://" + urlBase + "namespace=" + name

			//Emitimos al Emisor su url para consumir
			so.Emit("url", url)
		}

		//con so.On('NOMBRE_DEL_EVENTO', func()) hacemos el socket escuche informacion de dicho evento y procese la informacion que recibimos del mismo.
		//Recibimos la emicion y la enviamos a todos los consumidores correspondientes al namespace
		so.On("stream", func(image string) {
			eventAndBro := "stream-" + nsp.name
			so.BroadcastTo(eventAndBro, eventAndBro, image)
		})

		//Recibimos los mensajes del chat y reenviamos a la sala a cual pertenece
		so.On("chat", func(m string) {
			//TODO: Validar que el mensaje no contenga etiquetas HTML


			//userName para guardar el nombre de quien emite.
			var userName string
			//Tipo: 'Emisor' se guarda el nombre del Namespace
			userName = nsp.name

			//Tipo: 'consumer' se guarda el nombre del consumidor
			if tp == "consumer"{
				userName = nsp.consumers[so.Id()].name
			}

			//Creamos la data que enviaremos.
			data := make(map[string]interface{})
			data["name"] = userName
			data["message"] = m

			//Emit
			so.BroadcastTo("stream-" + nsp.name, "message-" + nsp.name, data)

		})

		//Manejamos la desconexiones
		so.On("disconnection", func() {
			if tp == "consumer"{
				log.Println("Se ha desconectado un Consumidor")
				//disminuimos el contador
				nsp.counter -= 1

				//Validamos que emisor no este vacio y notificamos al mismo la cantidad de consumidores.
				if nsp.emitter != nil {
					nsp.emitter.so.Emit("count-consume", nsp.counter)
				}
			} else {
				//Debemos eliminar el emisor y notificar a los consumidores del mismo
				log.Println("Se ha desconectado el emisor del namespace: " + nsp.name)
				so.BroadcastTo("stream-" + nsp.name, "streaming-closed", "closed")
			}

		})

	})

	//Imprimimos los errores del socket en caso que hayan.
	server.On("error", func(so socketio.Socket, err error) {
		log.Println("error: ", err)
	})

	//Le pasamos el server de Socketio como Handle a nuestra ruta '/socketio.io/'
	http.Handle("/socket.io/", server)

	//Utilizamos http.FileServer y le pasamos la carpeta donde estan los archivos Estáticos.
	http.Handle("/", http.FileServer(http.Dir("./public")))

	log.Println("Servidor corriendo en localhost:5001")

	//Levantamos el server con HTTPS utilizando certificados y llave privada del mismo.
	error := http.ListenAndServeTLS(":5001", "cert.pem", "key.pem", nil)
	if error != nil {
		log.Fatal("ListenAndServe: ", err)
	}

}
```

Como vemos en el codigo con	http.Handle("/", http.FileServer(http.Dir("./public"))) indicamos que vamos a servir los archivos estaticos que se encuentran en la carpeta public a la ruta raiz.

Creamos dicha carpeta
``mkdir public && cd public``
Dentro crearemos los archivos con lo siguiente:

## Archivo index.html
Simple index, que muestra un link a la página de emisor.

```html
<!DOCTYPE html>
<html lang="es">
  <head>
    <meta charset="UTF-8">
    <title>WebCam Live</title>
  </head>
  <body>
    <p>Streaming en Go con Socketio</p>
    <p><a href="emit.html">Ir a emitir</a></p>
  </body>
</html>
```

## Archivo emit.html

En el archivo "emit.html" el objeto menos común es 'navigator.getUserMedia'

Pide al usuario permiso para usar un dispositivo multimedia como una cámara o micrófono.
Si el usuario concede este permiso, el successCallback es invocado en la aplicación llamada con un objeto LocalMediaStream como argumento.

```js
//En este proyecto, solo solicite 'video'
navigator.getUserMedia ( { video: true, audio: true }, successCallback, errorCallback );
```

**successCallback**
La función getUserMedia llamará a la función especificada en el successCallback con el objeto LocalMediaStream que contenga la secuencia multimedia. Puedes asignar el objeto al elemento apropiado y trabajar con él.

**errorCallback**
La función getUserMedia llama a la función indicada en el errorCallback con un código como argumento.

```html
<html lang="es">
<head>
  <meta http-equiv="content-type" content="text/html; charset=utf-8">
    <link rel="stylesheet" href="css/init.css">
  <script src="https://code.jquery.com/jquery-1.12.4.min.js"   integrity="sha256-ZosEbRLbNQzLpnKIkEdrPv7lOy9C27hHQ+Xp8a4MxAQ="   crossorigin="anonymous"></script>
  <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/1.4.8/socket.io.js"></script>
  <title>Emit Video</title>
</head>
<body>
    <h2 class="center">Video</h2>
    <div class="box-namespace">
        <input type="text" id="namespace">
        <button onclick="initSocket()">Iniciar Streaming</button>
    </div>
    <video src="" id="video" autoplay="true"></video>
    <canvas id="preview" style="display:none;"></canvas>

    <div  id="box_logger">
        <div class="logger">
            <i id="box_logger-icon"></i>
            <spam id="logger"></spam>
        </div>
    </div>
    <div id="info" class="hidden">
        <div  class="isa_info hidden">
            <div class="logger">
                <i class="fa fa-info-circle"></i>
                Consumidores conectados: <spam id="counter"></spam>
            </div>
        </div>
        <div  class="isa_info hidden">
            <div class="logger">
                URL: <spam id="url"></spam>
            </div>
        </div>
        <div id="chat" class="center">
            <div id="box_chat" class="center"></div>
            <input class="center" type="text" id="message" placeholder="Escribe tu Mensaje">
        </div>
    </div>
  <script type="text/javascript" charset="utf-8">
    function initSocket(){

        //Obtenemos el namespace que indicamos en el input
        var namespace  = $("#namespace").val();

        //Instanciamos Socketio, pasando por QueryParam los parametros de conexion
        var socket = io.connect('', {query: 'type=emit&namespace=' + namespace });

        //Obtenemos el DIV donde se mostrara el video
        var video = document.getElementById("video");

        //Obtenemos el canvas y el contexto y fijamos su tamaño.
        var canvas = document.getElementById("preview");
        var context = canvas.getContext("2d");
        canvas.width = 400;
        canvas.height = 400;
        context.width = canvas.width;
        context.height = canvas.height;


        //Evento[count-consume] - Actualizamos el contador segun lso consumidores
        socket.on("count-consume", function (count) {
            console.log(count)
            $("#counter").text(count)
        });

        //Evento[url] - renderizamos la url que recibimos
        socket.on("url", function (url) {
            $("#url").text(url)
        });

        //Obtenemos el div de la caja de chat
        var $boxChat = $("#box_chat");

        //Evento[message-namespace] - renderizamos los mensajes que recibimos del socket
        socket.on('message-' + namespace, function(data){
            $boxChat.append("<p>" + data.name + ": "  + data.message + "</p>");
        });

        //Obtenemos el input del mensaje
        var $message = $("#message");
          //capturamos el evento keyup y validamos que sea la tecla ENTER
        $message.keyup(function(event){
            if(event.keyCode == 13){
                if ($message.val().match(/<(\w+)((?:\s+\w+(?:\s*=\s*(?:(?:"[^"]*")|(?:'[^']*')|[^>\s]+))?)*)\s*(\/?)>/)) {
                    alert('No puedes utilizar HTML');
                    $message.val("")
                } else {
                    //Obtenemos el valor del input
                    var message = $($message).val();
                    //Realizamos el append a la caja
                    $boxChat.append("<p> Yo: "   + message + "</p>");
                    //Volvemos a poner en blanco el valor del input
                    $message.val("")
                    //Emitimos el mensaje
                    socket.emit("chat", message);
                }
            }
        });

        //Obtenemos el getUserMedia segun el navegador
        navigator.getUserMedia = (navigator.getUserMedia || navigator.webkitGetUserMedia
        || navigator.mozGetUserMedia || navigator.msgGetUserMedia);

        if(navigator.getUserMedia){
            navigator.getUserMedia({video:true}, ok, fail);
        }

        //Se encarga de ejecutar la function Consume para realizar el proceso.
        setInterval(function(){
            Consume(video, context)
        }, 100);

        //funcion que se encarga de los mensajes
        function logger(type, msg){
            if (type == 'sucess'){
                $("#box_logger").addClass("isa_success")
                $("#box_logger-icon").addClass("fa fa-check")
            }else{
                $("#box_logger").addClass("isa_error")
                $("#box_logger-icon").addClass("fa fa-times-circle")
            }
            var info = $("#info")
            info.removeClass("hidden")
            info.addClass("show")
            $("#logger").text(msg);
        }

        //successCallback que le pasamos a getUserMedia
        function ok(stream){
            video.src = window.URL.createObjectURL(stream)
            logger('sucess', 'Camara disponible !');
        }

        //errorCallback que le pasamos a getUserMedia
        function fail(){
            logger('error', 'Ha ocurrido un problema, No logramos detectar su Camara !');
        }


        function Consume(video, context){
            context.drawImage(video, 0, 0, context.width, context.height);
            socket.emit('stream', canvas.toDataURL('image/webp'));
        }

    }
  </script>
</body>
</html>
```

## Archivo consume.html

```html
<html>
<head>
    <meta http-equiv="content-type" content="text/html; charset=utf-8">
    <link rel="stylesheet" href="css/init.css">
    <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/1.4.8/socket.io.js"></script>
    <script src="https://code.jquery.com/jquery-1.12.4.min.js"   integrity="sha256-ZosEbRLbNQzLpnKIkEdrPv7lOy9C27hHQ+Xp8a4MxAQ="   crossorigin="anonymous"></script>
    <title>Consume stream</title>
</head>
<body>
<div class="stream" style="text-align: center">
  <h2>Streaming</h2>
  <img id="play" style="border: solid 1px;" >
  <div id="chat" class="center">
      <div  id="box-init" class="box-namespace">
          <input type="text" id="name" placeholder="Tu nombre">
          <button onclick="initSocket()" >Ver Streaming</button>
      </div>
      <div id="container" class="hidden">
          <div id="box_chat" class="center"></div>
          <input class="center" type="text" id="message" placeholder="Escribe tu Mensaje">
      </div>
  </div>
</div>

  <script type="text/javascript" charset="utf-8">
      //func para obtener los parametros desde la URL
      var QueryString = function () {
      // This function is anonymous, is executed immediately and
      // the return value is assigned to QueryString!
      var query_string = {};
      var query = window.location.search.substring(1);
      var vars = query.split("&");
      for (var i=0;i<vars.length;i++) {
        var pair = vars[i].split("=");
        // If first entry with this name
        if (typeof query_string[pair[0]] === "undefined") {
          query_string[pair[0]] = decodeURIComponent(pair[1]);
          // If second entry with this name
        } else if (typeof query_string[pair[0]] === "string") {
          var arr = [ query_string[pair[0]],decodeURIComponent(pair[1]) ];
          query_string[pair[0]] = arr;
          // If third or later entry with this name
        } else {
          query_string[pair[0]].push(decodeURIComponent(pair[1]));
        }
      }
      return query_string;
    }();

      function initSocket(){
          if (QueryString.namespace){
              //Utilizamos la func QueryString para obtener el namespace de la URL
              var namespace = QueryString.namespace

              //Obtenemos el nombre del consumidor
              var user = $("#name").val()

              //Instanciamos Socketio, pasando por QueryParam los parametros de conexion
              var socket = io.connect('', {query: 'type=consumer&namespace=' + namespace + '&user=' + user, reconnection: false});

              socket.on('stream-' + namespace, function(image){
                  var img = document.getElementById("play");
                  img.src = image;
              });

              $("#container").removeClass("hidden");
              $("#container").addClass("show");
              $("#box-init").addClass("hidden");

              //Obtenemos el div de la caja de chat
              var $boxChat = $("#box_chat");

              //Evento[message-namespace] - renderizamos los mensajes que recibimos del socket
              socket.on('message-' + namespace, function(data){
                  $boxChat.append("<p>" + data.name + ": "  + data.message + "</p>");
              });

              //Obtenemos el input del mensaje
              var $message = $("#message");
        //capturamos el evento keyup y validamos que sea la tecla ENTER
        $message.keyup(function(event){
            if(event.keyCode == 13){
                if ($message.val().match(/<(\w+)((?:\s+\w+(?:\s*=\s*(?:(?:"[^"]*")|(?:'[^']*')|[^>\s]+))?)*)\s*(\/?)>/)) {
                    alert('No puedes utilizar HTML');
                    $message.val("")
                } else {
                    //Obtenemos el valor del input
                    var message = $($message).val();
                    //Realizamos el append a la caja
                    $boxChat.append("<p> Yo: "   + message + "</p>");
                    //Volvemos a poner en blanco el valor del input
                    $message.val("")
                    //Emitimos el mensaje
                    socket.emit("chat", message);
                }
            }
        });
          }

      };
  </script>
</body>
</html>
```

## Archivo init.css

Como veran en las imagenes de ejemplo este proyecto tiene muy poco css, debido a que solo necesitaba lo minimo para mostrar algo funcional y que se lograra apreciar y porque realmente no soy muy bueno con el css jajaja.


```css
/* Tengo como 1 año sin hacer nada de  CSS */
@import url('//maxcdn.bootstrapcdn.com/font-awesome/4.3.0/css/font-awesome.min.css');

#video {
    margin-left: auto;
    margin-right: auto;
    display: block
}

.center{
    text-align: center;
}

.logger{
    margin-left: -4px;
    margin-top: -8px;
}
.isa_info, .isa_success, .isa_error {
    height: 15px;
    width: 615px;
    padding:12px;
    margin-left: auto;
    margin-right: auto;
    display: block
}
.isa_info {
    color: #00529B;
    background-color: #BDE5F8;
}
.isa_success {
    color: #4F8A10;
    background-color: #DFF2BF;
}
.isa_error {
    color: #D8000C;
    background-color: #FFBABA;
}
.isa_info i, .isa_success i,  .isa_error i {
    font-size:2em;
    vertical-align:middle;
}
```


Este es el primer ariticulo que escribo de muchos "eso espero", la verdad me costo bastante hacerlo a pesar que es corto, pero como desarrolladores debemos poder compartir conocimiento con la comunidad y que mejor forma de hacerlo que empezando con pequeños artículos, los invito a que se animen y realicen sus propios artículos para compartir contenido.

Para los que deseen ver el repositorio y contribuir o hacer alguna crítica :D
Github: [https://github.com/douglasmakey/streaming-go-socketio](https://github.com/douglasmakey/streaming-go-socketio)