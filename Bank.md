# Bank

![bank](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/fbce3aac-a153-48fa-914b-4a8ca96d4a87)


## Fase de reconocimiento

En primer lugar, realizamos un ping -c1 <ip> a la ip de la maquina para verificar conectividad. Como observamos, el ttl de respuesta del ping posee el numero 63, es decir que probablemente estemos ante un sistema Linux (ttl 64). 

Arrancamos haciendo un escaner de todos los puertos utilizando nmap con los parametros en cuestion para agilizar el escaneo.

![sc1](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/d19880b7-03a8-4744-9687-22b3932338e0)

Como podemos ver, la maquina posee 3 puertos abiertos. El puerto 22 de ssh, el 53 de DNS y el 80 que posiblemente este hosteando una pagina web.

Una vez hecho el escaneo general, procedemos a realizar el escaneo mas intensivo en estos puertos abiertos.

![sc2](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/a9f386b6-b18b-4434-9b27-783d75ac0fb7)

Si ingresamos desde el navegador a la direccion de la maquina victima podemos ver efectivamente la pagina default de Apache2 de Ubuntu.

Procedemos a hacer directory busting con la herramienta dirsearch.

![sc3](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/d693176a-61dd-44d3-9516-3892c7ce38df)

Como podemos ver no se encuentran ningunas rutas interesantes que sean accesibles para nosotros. Esto es un indicio de que debemos probar por otro lado.

Vemos que el puerto 53 pertenece a DNS, por lo que podria haber una transferencia de zona. En principio no tenemos el nombre de dominio, solo la ip de la maquina victima, pero como el nombre de la maquina es Bank, probamos usando bank.htb con el comando dig y obtenemos la siguiente informacion:

![sc4](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/4927a395-0e0f-4ef0-9851-2826a658a6db)


En base a esas respuestas del comando dig, probamos añadiendolas en el archivo /etc/hosts con la ip en cuestion.

Buscando con el navegador cada una de las entradas, en todas nos redirige nuevamente a la pagina default de apache, salvo una: bank.htb. Esta pagina nos redirige a un login.php

![sc5](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/80953e94-177e-4644-b2a5-ea5815cb7bb8)

Nuevamente hacemos directory listing y encontramos directorios /assests, /inc y /balance-transfer.

En /assets y /inc encontramos la posibilidad de hacer directory listing pero aparte de eso no hay nada interesante. En cambio, en /balance-transfer encontramos lo que parecen ser registros de transacciones

![sc6](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/7f34ab29-c62f-42a0-aa1e-9abd6c192f19)

La mayoria tiene el mismo tamaño y representan una encripcion exitosa de usuario y contraseña, pero si la ordenamos segun el size en forma ascendente podemos ver un archivo que resalta:

![sc7](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/78b6c838-c699-4136-812c-b2e5175a53f5)

El tamaño del archivo menor al resto se debe a que, como vemos, el encrypt fallo por lo tanto los datos estan expuestos y ocupan menor espacio.

Al obtener las credenciales, las probamos en la pagina del login.php e ingresamos a la cuenta!

![sc8](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/1468bfb2-9aa1-4690-9f33-9b119dcd0ef0)


## Explotacion

En la seccion de support la pagina nos da la opcion de crear un ticket y subir una imagen. 

En este punto, el sistema no deja subir archivos que tengan como extension .php. Al ver esto, una idea es agregarle la extension .png al final de nuestro file para bypassear esta condicion y lo logramos.

Sin embargo, al navegar a /uploads/nuestro_file.php.png no logramos que se ejecute el script en php ya que el sistema lo toma como una imagen.

![sc9](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/91370406-fc01-40f8-a0d0-0ce66160bd88)


Seguimos investigando por la pagina, y logramos ver un dato muy importante en el page source del support.php :

![sc10](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/8f77ee60-22b7-4091-a38f-e8e3d219df51)


Esto quiere decir que si subimos nuestro archivo con extension .htb vamos a lograr ejecutarlo como codigo php.

En base a esto, subimos nuestro archivo malicioso y logramos obtener una reverse shell.

![sc11](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/0673d0c9-dfe8-4b22-b245-23b03e7933e2)


## Escalar privilegios

Investigando por el sistema, encontramos una importante falla de seguridad.

![sc12](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/07597836-b897-4ff8-a06e-ff8293d91395)


Esto quiere decir que el archivo /etc/passwd, que es donde se almacenan los users, es writable para cualquier usuario!! 

Al tener esta posibilidad, es tan facil como editar dicho archivo eliminandole el placeholder “x” de la password de root y escribiendo una nueva password previamente hasheada con el comando openssl passwd <nueva_password>. 

![sc13](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/66467d85-349d-43e4-9642-813d2c3166be)


Esto hace que luego nos podamos loguear con estas credenciales y ya somos root!

![sc14](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/b1d9d09d-8a76-442f-8350-c587ede36cfd)


### Pwned!
