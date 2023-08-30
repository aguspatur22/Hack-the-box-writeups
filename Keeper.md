# Keeper
![a1](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/02ef2821-f661-4af5-933f-9f59459c6fb9)

## Fase de reconocimiento

En primer lugar, realizamos un ping -c1 <ip> a la ip de la maquina para verificar conectividad. Como observamos, el ttl de respuesta del ping posee el numero 63, es decir que probablemente estemos ante un sistema Linux (ttl 64). 

Arrancamos haciendo un escaner de todos los puertos utilizando nmap con los parametros en cuestion para agilizar el escaneo.

La maquina posee 3 puertos abiertos: 22, 80 y 8000. Un detalle que hay que remarcar es que yo suelo ejecutar este escaneo como minimo 3 veces, ya que puede ser que en los primeros no nos detecte todos los puertos abiertos. En esta maquina en el primer escaneo solo marcaba el 22 y el 80, y solo a partir del segundo y tercer escaneo salto el puerto 8000.

Procedemos a realizar un descubrimiento de servicios y versiones mas detallado en los puertos en cuestion:

![a2](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/2976ff6b-8266-447a-bfda-3d84452bf67c)

A primera vista podemos ver que posee 2 servicios web en los puertos 80 y 8000 respectivamente. El puerto de SSH no nos interesa de momento ya que la unica forma de explotacion de esa version de OpenSSH seria mediante brute-force attack.

Usamos el navegador para investigar las 2 webs en cuestion:

En el puerto 80:

![a3](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/5803337e-a8d4-4c08-afe0-a85b8f52da55)

Para poder ingresar a esa pagina, debemos agregar la ip de la maquina y esa url a nuestro archivo /etc/hosts. 

![a4](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/5be93300-7c34-423e-b6af-aeec0fa92ea6)

Parece ser un login para el cual por ahora no tenemos credenciales. Un dato que nos podria llegar a servir a futuro es la version del Request Tracker (4.4.4) para buscar vulnerabilidades y tambien la version del sistema operativo Ubuntu para una posterior escalada de privilegios.

## Explotación

En base a la informacion de versionado de la web recogida en la etapa anterior, probamos suerte buscando en nuestro navegador default credentials para este servicio y las encontramos: user:root y password:password.

![a5](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/7b6d4267-9d87-45fa-853e-5cdce56c6a04)

Logramos ingresar como usuario root!

Navegando por la interfaz podemos ver que parece ser un sistema de tickets a resolver. Unicamente podemos visualizar uno, el que posee id 300000 creado por Owner:[lnorgaard (Lise Nørgaard)](http://tickets.keeper.htb/rt/User/Summary.html?id=27). 

Leyendo se da a entender que habia un problema con el cliente de Keepass (es un password manager que te permite guardar credenciales en una unica bd con una master key). Se habla de un archivo en cuestion pero que fue borrado por temas de seguridad.

Investigando un poco mas la interfaz, y aprovechando que somos root, podemos ver una seccion de administracion de usuarios, en la cual encontramos lo siguiente:

![a6](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/dbe1bdd2-dd8f-41e9-8604-67cc2c1fe50f)

Para probar si esa es la password actual, nos deslogueamos como root e iniciamos sesion como el usuario lnorgaard y vemos que si!

Si nos acordamos del escaneo inicial de NMAP podemos ver que teniamos el puerto 8000 que nos permito directory listing de la ubicacion “/” pero en la mayoria no podemos ver archivos interesantes, y los que si lo son, no nos deja descargarlos.

El otro puerto era SSH, asique vamos a probar conectandonos con el usuario lnorgaard y la password Welcome2023!

![a7](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/8a171ecb-688f-42d7-86ce-d18b0f7ced6d)

## Escalar privilegios

Dentro del usuario lnorgaard, podemos ver un archivo .zip que si lo descomprimimos nos da el archivo KeePassDumpFull.dmp y el passcodes.kdbx

La version del keepass es vulnerable al siguiente CVE:

![a8](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/881d96c0-2e30-4de9-9f3c-0b5c5e1bc13c)

Utilizando este PoC (https://github.com/CMEPW/keepass-dump-masterkey) logramos obtener una contraseña parcial:

![a9](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/286b618c-ae31-4438-84b0-8beef72eac3a)

Al principio la password no resulta familiar, pero con una busqueda en google y teniendo en cuenta que el usuario lnorgaard en cuestion tenia el lenguaje del servidor web en danes, damos con esta frase en particular: **Rødgrød Med Fløde** 

Utilizamos esa password para acceder al keepass con el cliente “kpcli”.

![a10](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/b4f1cdc0-7787-4616-a03c-7f5222924d05)

Utilizando la opcion -f nos muestra la password, la cual si probamos conectandonos por ssh nos resulta incorrecta.

En base a las notas de ese archivo vemos que es una clave ssh-rsa del tipo PuTTY, la cual podemos usar para conectarnos por el puerto 22 como root!

Antes de eso, debemos poner el contenido de esa PuTTY key en un file, y convertirlo a .pem. Luego, nos conectamos por ssh con la key en cuestion y no necesitamos proveer contraseña!!

![a11](https://github.com/aguspatur22/Hack-the-box-writeups/assets/50930830/da5c228b-bf27-42b3-b02c-ecb3183953c2)

### Pwned!
