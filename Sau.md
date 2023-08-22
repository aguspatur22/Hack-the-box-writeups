# Sau

![s1](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/4bad384a-257c-4cc6-bf1d-49ba5b5a8eef)

## Fase de reconocimiento

Para empezar, realizamos un ping -c1 <ip> a la maquina victima para probar conectividad. Vemos que nos responde asique podemos proceder. Un detalle que vale destacar es que en la respuesta el TTL tiene un valor de 63, esto quiere decir que estamos ante una maquina Linux muy posiblemente (asociadas con TTL 64).

Realizamos un escaneo de todos los puertos de la maquina y vemos que estan abiertos el 22 y el 55555. Como ya sabemos, el 22 es de ssh pero el otro puerto en principio es desconocido. 

En base a la informacion de los puertos abiertos, ahora hacemos un escaneo exhaustivo con el parametro de -sCV en Nmap para encontrar versiones de servicios y detección de scripts de seguridad.

![s2](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/0f0632b3-558b-44c6-a8ed-b8c47bd03468)

El resultado nos arroja que existe una pagina web en el puerto 55555 que desconociamos, mas precisamente en <ipMaquinaVictima>:55555/web

![s3](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/64f87c5a-8071-468a-9d3b-067df2edb8e2)

Buscando en internet encontramos que "Request Baskets" es una herramienta que permite recolectar y analizar solicitudes HTTP. Está diseñada para probar y depurar webhooks, notificaciones, clientes REST y otros tipos de solicitudes web.

Cual es la utilidad? Imaginense que estan desarrollando una aplicación que recibe notificaciones de terceros a través de webhooks. Con Request Baskets, pueden crear un "cesto de solicitudes" donde las solicitudes HTTP entrantes se recopilan y muestran de manera organizada. Esto te permite verificar si las solicitudes llegan correctamente, examinar los datos enviados y comprobar si todo funciona como se espera.

Vemos en la parte de abajo de la pagina que la version utilizada es la 1.2.1. Esto representa una gran brecha de seguridad debido a que ahora al conocer la version podriamos buscar exploits potenciales para este servicio.

## Explotacion

Con esa informacion, y una rapida busqueda en internet, encontramos que es vulnerable a una "Server-Side Request Forgery" (SSRF), lo que significa que los atacantes pueden manipular solicitudes desde el servidor para acceder a recursos de red e información confidencial (CVE-2023-27163).

Siguiendo el formato del CVE, podemos aprovechar la vulnerabilidad en /api/baskets/{basket_name}. Realizamos una prueba rapida utilizando [google.com](http://google.com) para ver si realiza la request y nos lleva a esa pagina. 

Un detalle MUY importante, es que la opcion de “proxy_response” este seteada en true, de lo contrario la maquina no va a hacer efectiva al consulta a la url en cuestion (el valor en el resto de campos es indiferente).

Al realizar la consulta con el curl y posteriormente dirigirnos a http://<ipMaquinaVictima>:55555/<basket_name> vemos lo siguiente:

![s4](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/212b418f-68b9-4f1d-94e7-5c24a3f0d446)

Esto quiere decir que efectivamente es vulnerable a SSRF. 

Ahora, vamos a crear un script que nos de la posibilidad de probar multiples puertos que nosotros le pasemos para ver si logramos dar con un servicio que este habilitado en la red interna pero no este expuesto al exterior por ej mediante un firewall (razon por la cual no lo hayamos podido visualizar con el escaneo de Nmap).

```jsx
#!/bin/bash
# Agustin Paturlanne https://github.com/aguspatur22

# Verificar argumentos
if [ $# -lt 2 ]; then
	echo "Uso: $0 <URL> <port1> [<port2>...] "
	exit 1
fi

# Obtenemos url del primer argumento
url="$1"
shift 1

# Obtenemos lista de puertos
ports=("$@")

# Iteracion y consultas
for port in "${ports[@]}"; do
	curl -s --location "$url/api/baskets/port$port" --header "Content-Type: application/json" --data '{"forward_url": "http://127.0.0.1:'$port'/", "proxy_response": true,	"insecure_tls": false, "expand_path":false, "capacity":200}' > /dev/null

	if [ "$(curl -s -I "$url/port$port" | grep "HTTP" | awk -F" " '{print $2}')" -eq 501 ] ; then
		echo "Server encontrado en $url/port$port"
	else
		echo "No se encontró server en $url/port$port"
	fi
done
```

Probamos dicho script con los puertos que suelen alojar servicios web y encontramos una coincidencia: 

![s5](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/f43a9e29-b577-4300-a34d-9b2479623907)

Navegamos hacia esa url:

![s6](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/75bfc77b-9556-48db-b109-bb98f456710f)

Bingo! Encontramos Mailtrail que, en base a la documentacion, es un sistema de deteccion de trafico malicioso (https://github.com/stamparm/maltrail/blob/master/README.md#introduction).

Afortunadamente para nosotros, podemos visualizar que en la misma pagina nos dice la version de Mailtrail, ahora queda realizar una rapida busqueda para averiguar si hay exploits relacionados con este servicio: https://github.com/spookier/Maltrail-v0.53-Exploit

Este exploit es bastante simple, lo que hace es aprovechar el uso de “;”.Debido a una funcion que chequea el input del usuario y ejecuta un comando a nivel de sistema, podemos aprovecharnos de esto y ingresar el usuario a loguear y luego el caracter ; para despues enviar otros comandos a ejecutar.

Como el exploit va dirigido al login y la pagina no posee el link al /login, no podemos ingresar arbitrariamente a esa url porque acordemosnos que es un servicio interno restringido por lo cual no va a funcionar. Entonces, debemos actualizar nuevamente los parametros del basket en cuestion para que en vez de que nos rediriga a http://127.0.0.1:80/ nos lleve a http://127.0.0.1:80/login entonces ahi podemos ejecutar correctamente el exploit.

![s7](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/9e07dab1-48e4-4816-a368-082d000fed3f)

Conseguimos la shell!

## Escalar privilegios

Ya adentro de la shell podemos hacer un tratamiento de la misma para tener todos las comodidades del teclado.

Una vez hecho esto, usamos el comando find / -name user.txt 2>/dev/null para encontrar la user flag.

![s8](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/39fa8cf9-accf-46be-90d9-cf3c4b4bbed7)

Ahora, ejecutamos sudo -l para ver si tenemos algun privilegio y vemos lo siguiente:

![s9](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/b10e8c7d-b66c-47a0-b47d-c6ca6292f917)

Esto quiere decir que podemos ejecutar ese binario como si fueramos root, el cual resulta ser el sistema de deteccion de trafico malicioso - Maltrail.

![s10](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/3bd2c93a-f454-4c16-813e-a9b345af439d)

Con una busqueda rapida en GTFObins, podemos ver que generalmente el systemctl invoca al pager less, que directamente ahi al ejecutarlo como sudo podemos spawnearnos una shell!

![s11](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/4420578d-85b4-4475-bc71-135f02d4c67b)

### Pwned!
