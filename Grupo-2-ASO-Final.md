1) Preparación básica en el servidor (Debian 13) 

Actualiza sistema y instala dependencias útiles: 

# Como root o con sudo 
sudo apt update 
sudo apt upgrade -y 
sudo apt install -y wget curl openssl apt-transport-https ca-certificates gnupg ufw fail2ban 
 

(Comprueba que el servidor tiene conectividad con el cliente/lan). 

 

2) Instalar Webmin (repositorio oficial .deb) 

Método robusto (repositorio oficial Webmin): 

# Añadir la clave GPG del repositorio Webmin 
wget -qO- http://www.webmin.com/jcameron-key.asc | sudo gpg --dearmor -o /usr/share/keyrings/webmin-archive-keyring.gpg 
 
# Añadir el repo a sources.list.d 
echo "deb [signed-by=/usr/share/keyrings/webmin-archive-keyring.gpg] http://download.webmin.com/download/repository sarge contrib" | sudo tee /etc/apt/sources.list.d/webmin.list 
 
# Actualizar e instalar webmin 
sudo apt update 
sudo apt install -y webmin 
 

Comprobar estado del servicio: 

sudo systemctl status webmin --no-pager 
sudo systemctl enable --now webmin 
 

Webmin por defecto escucha en el puerto 10000 y con SSL (miniserv). Pero vamos a endurecer/configurar. 

 

3) Crear usuario administrador Webmin desde la CLI 

Crea un usuario Webmin (ej. webadmin) y contraseña segura (hazlo interactivo o en una sola línea): 

# Cambiar la contraseña del usuario root de Webmin (o crear uno nuevo) 
# Script incluido con webmin: changepass.pl 
sudo /usr/share/webmin/changepass.pl /etc/webmin webadmin 'TuPasswordSeguraAqui' 
 

Nota: sustituye 'TuPasswordSeguraAqui' por la contraseña real. Si quieres pedir al script que te pregunte la contraseña, omite el último argumento. 

 

4) Generar/instalar certificado TLS (self-signed) y forzar HTTPS 

En una red local sin dominio público es habitual usar un certificado autofirmado. Generamos uno y lo integramos en Webmin. 

# Crear carpeta temporal 
cd /tmp 
 
# Generar clave privada y certificado autofirmado (válido 365 días) 
sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 \ 
 -keyout webmin.key -out webmin.crt \ 
 -subj "/C=ES/ST=Madrid/L=Madrid/O=MiOrg/OU=IT/CN=192.168.33.10" 
 
# Combinar en el formato que usa Webmin (miniserv.pem contiene clave+cert) 
sudo bash -c "cat webmin.key webmin.crt > /etc/webmin/miniserv.pem" 
 
# Ajustar permisos 
sudo chown root:root /etc/webmin/miniserv.pem 
sudo chmod 600 /etc/webmin/miniserv.pem 
 

Forzar Webmin a usar TLS, bind a la IP del servidor y (si quieres) cambiar el puerto: 

# Asegurar ssl está activo y bind en la IP del servidor 
sudo sed -i 's/^ssl=.*/ssl=1/' /etc/webmin/miniserv.conf || echo "ssl=1" | sudo tee -a /etc/webmin/miniserv.conf 
# Forzar enlace a IP 
sudo sed -i 's/^bind=.*/bind=192.168.33.10/' /etc/webmin/miniserv.conf || echo "bind=192.168.33.10" | sudo tee -a /etc/webmin/miniserv.conf 
# (Opcional) cambiar el puerto a 10443 por ejemplo 
sudo sed -i 's/^port=.*/port=10443/' /etc/webmin/miniserv.conf || echo "port=10443" | sudo tee -a /etc/webmin/miniserv.conf 
 
# Reiniciar webmin 
sudo systemctl restart webmin 
 

Ahora Webmin escuchará en 192.168.33.10:10443 (si usaste 10443) o en :10000 si no cambiaste el puerto. Ajusta el puerto según tus políticas. 

 

5) Firewall: permitir sólo al cliente 192.168.33.100 y bloquear resto (UFW) 

Si quieres que sólo la IP cliente acceda a Webmin: 

# Activar UFW (si no lo está) 
sudo ufw allow OpenSSH   # permitir SSH si lo necesitas 
sudo ufw enable 
 
# Permitir solo desde 192.168.33.100 al puerto (usa 10443 o 10000 según tu config) 
# Ejemplo con puerto 10443: 
sudo ufw allow from 192.168.33.100 to 192.168.33.10 port 10443 proto tcp 
 
# Denegar acceso al puerto desde otras IPs (UFW por defecto lo hará si no hay otra regla) 
sudo ufw status numbered 
 

Si no estás usando UFW sino iptables/nftables, puedes crear regla equivalente. Con UFW te aseguras restricción sencilla. 

 

6) Hardening adicional: Fail2ban para Webmin 

Instalar y activar fail2ban con filtro para detectar intentos de login en Webmin. 

sudo apt install -y fail2ban 
 
# Crear jail local para webmin 
sudo tee /etc/fail2ban/jail.d/webmin.local > /dev/null <<EOF 
[webmin] 
enabled = true 
port = 10443 
filter = webmin 
logpath = /var/webmin/miniserv.log 
maxretry = 5 
bantime = 3600 
findtime = 600 
EOF 
 
# Crear filtro: /etc/fail2ban/filter.d/webmin.conf 
sudo tee /etc/fail2ban/filter.d/webmin.conf > /dev/null <<'EOF' 
[Definition] 
failregex = Authentication failed for .* from <HOST> 
ignoreregex = 
EOF 
 
# Reiniciar fail2ban 
sudo systemctl restart fail2ban 
sudo systemctl enable fail2ban 
 

(Revisa los paths de logs si difieren: /var/webmin/miniserv.log es común.) 

 

7) Restricción de módulos / acceso por usuario en Webmin 

Desde Webmin UI -> Webmin Users puedes limitar módulos. En CLI puedes editar /etc/webmin/miniserv.conf y /etc/webmin/webmin.acl según documentación, pero lo más rápido es usar la interfaz web para asignar permisos al usuario webadmin. 

 

8) Subir Documentacion.md y Documentacion.pdf desde el cliente (192.168.33.100) 

Opción A — SCP (desde el cliente): 

# Desde cliente 192.168.33.100 en su terminal: 
scp Documentacion.md Documentacion.pdf [usuario_del_servidor@192.168.33.10:/home/usuario_del_servidor/webdocs/](mailto:usuario_del_servidor@192.168.33.10:/home/usuario_del_servidor/webdocs/) 
# ejemplo: 
scp Documentacion.* [alumno@192.168.33.10:/home/alumno/webdocs/](mailto:alumno@192.168.33.10:/home/alumno/webdocs/) 
 

Asegúrate de que la carpeta /home/alumno/webdocs/ exista: 

# En servidor crear carpeta y ajustar permisos 
sudo mkdir -p /home/alumno/webdocs 
sudo chown alumno:alumno /home/alumno/webdocs 
 

Opción B — Usar el File Manager de Webmin: 

En el navegador del cliente abrir: https://192.168.33.10:10443/ 

Login con webadmin y la contraseña. 

Ir a Other -> File Manager y usar "Upload" para subir los archivos a /home/alumno/webdocs/. 

 

9) Gestionar servicios desde Webmin (ejemplos y comandos equivalentes CLI) 

Webmin permite administrar servicios en la sección System -> Bootup and Shutdown o System -> System Services. Aquí los equivalentes por CLI: 

Reiniciar cups (o cualquier demonio de prueba): 

# Desde servidor 
sudo systemctl restart cups 
sudo systemctl status cups --no-pager 
# O con service 
sudo service cups restart 
 

Iniciar/detener Apache/Nginx: 

sudo systemctl restart apache2 
sudo systemctl restart nginx 
 

Desde Webmin: Modules → System Services → seleccionar servicio → Stop/Start/Restart. 

 

10) Ejemplo de un flujo completo (instalar Webmin, subir doc, reiniciar servicio) — todos los pasos en bruto 

Resumen de comandos encadenados (ejecutar en el servidor): 

# Actualizar e instalar prereqs 
sudo apt update && sudo apt upgrade -y 
sudo apt install -y wget openssl ufw fail2ban 
 
# Añadir repo webmin e instalar 
wget -qO- http://www.webmin.com/jcameron-key.asc | sudo gpg --dearmor -o /usr/share/keyrings/webmin-archive-keyring.gpg 
echo "deb [signed-by=/usr/share/keyrings/webmin-archive-keyring.gpg] http://download.webmin.com/download/repository sarge contrib" | sudo tee /etc/apt/sources.list.d/webmin.list 
sudo apt update 
sudo apt install -y webmin 
sudo systemctl enable --now webmin 
 
# Crear usuario webmin 
sudo /usr/share/webmin/changepass.pl /etc/webmin webadmin 'MiPassMuySegura123!' 
 
# Crear certificado self-signed y ponerlo en miniserv.pem 
sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 \ 
 -keyout /tmp/webmin.key -out /tmp/webmin.crt -subj "/C=ES/ST=Madrid/L=Madrid/O=Proyecto/OU=IT/CN=192.168.33.10" 
sudo bash -c "cat /tmp/webmin.key /tmp/webmin.crt > /etc/webmin/miniserv.pem" 
sudo chown root:root /etc/webmin/miniserv.pem 
sudo chmod 600 /etc/webmin/miniserv.pem 
 
# Forzar bind y puerto 
sudo sed -i 's/^bind=.*/bind=192.168.33.10/' /etc/webmin/miniserv.conf || echo "bind=192.168.33.10" | sudo tee -a /etc/webmin/miniserv.conf 
sudo sed -i 's/^port=.*/port=10443/' /etc/webmin/miniserv.conf || echo "port=10443" | sudo tee -a /etc/webmin/miniserv.conf 
sudo sed -i 's/^ssl=.*/ssl=1/' /etc/webmin/miniserv.conf || echo "ssl=1" | sudo tee -a /etc/webmin/miniserv.conf 
 
# Reiniciar webmin 
sudo systemctl restart webmin 
 
# Firewall: permitir solo cliente 
sudo apt install -y ufw 
sudo ufw allow OpenSSH 
sudo ufw enable 
sudo ufw allow from 192.168.33.100 to 192.168.33.10 port 10443 proto tcp 
sudo ufw status verbose 
 
# Instalar fail2ban y crear jail básico 
sudo apt install -y fail2ban 
sudo tee /etc/fail2ban/jail.d/webmin.local > /dev/null <<EOF 
[webmin] 
enabled = true 
port = 10443 
filter = webmin 
logpath = /var/webmin/miniserv.log 
maxretry = 5 
EOF 
sudo tee /etc/fail2ban/filter.d/webmin.conf > /dev/null <<'EOF' 
[Definition] 
failregex = Authentication failed for .* from <HOST> 
EOF 
sudo systemctl restart fail2ban 
 
# Crear carpeta para documentación 
sudo mkdir -p /home/alumno/webdocs 
sudo chown alumno:alumno /home/alumno/webdocs 
 

Desde el cliente, subir documentos: 

scp Documentacion.* [alumno@192.168.33.10:/home/alumno/webdocs/](mailto:alumno@192.168.33.10:/home/alumno/webdocs/) 
 

Acceder (cliente, en navegador): 

https://192.168.33.10:10443/ 
 

Aceptarás certificado autofirmado la primera vez. 

 

 

11) Solución de problemas comunes 

No se puede acceder al puerto 10443: sudo ss -tlnp | grep 10443 para ver si miniserv está escuchando. Revisa /var/log/syslog y /var/webmin/miniserv.log. 

Usuario/contraseña Webmin incorrecto: sudo /usr/share/webmin/changepass.pl /etc/webmin webadmin 'NuevaPass'. 

Fail2ban bloquea cliente por error: sudo fail2ban-client status webmin y sudo fail2ban-client set webmin unbanip 192.168.33.100. 

Permisos de miniserv.pem: deben ser root:root y 600.


2. PBI 2: Implementación de SSH y Cifrado (CE c, d, h):
   <h1> ◦ Tarea 2.1: Instalar y configurar el servidor OpenSSH (openssh-server) en Debian 13.</h1>


Voy a realizar esta parte del proyecto con un Debian 13 y un Server Debian .

<img width="1918" height="1080" alt="Image" src="https://github.com/user-attachments/assets/27a61232-ecf2-46c8-9485-98dcb85f1272" />

En esta imagen podemos ver la configuracion de red y que comparten espacio en la misma red

Asi que comenzaremos instalando SSH en los dos dispositivos.

Servidor:

<img width="1058" height="191" alt="Image" src="https://github.com/user-attachments/assets/c1ea70eb-0e95-44aa-8401-536de32eec18" />

Cliente:

<img width="753" height="177" alt="Image" src="https://github.com/user-attachments/assets/47ad4208-0ed3-4d8b-8f60-6202f2e0e874" />


Ahora configurare los archivos de ssh del servidor para que solo pueda acceder el cliente.

<img width="1318" height="1080" alt="Image" src="https://github.com/user-attachments/assets/658f5ab9-f1a5-4abb-b5c1-da10427e3f52" />


Lo haremos para que solo pueda entrar la IP del cliente .

Y ahora procederemos a bloquear las demas IPS's para hacer un entorno seguro.



<img width="1323" height="1080" alt="Image" src="https://github.com/user-attachments/assets/e324825a-7830-490c-8c99-86331faf6e75" />



Y de seguido reiniciamos el servicio para efectuar los cambios.

<img width="1320" height="1080" alt="Image" src="https://github.com/user-attachments/assets/a5f3ae58-4000-4612-9977-849d3e82b1f1" />

    <h1>◦ Tarea 2.2: Configurar sshd_config para utilizar exclusivamente el Protocolo SSH2 y deshabilitar o restringir el acceso root (criterio de seguridad).</h1>

Accedemos al archivo de configuracion en /etc/ssh/sshd_config , modificamos 2 lineas , la primera para que no puedan acceder con el usuario Root , y la segunda para que se use exclusivamente el protocolo de ssh2

<img width="1317" height="1079" alt="Image" src="https://github.com/user-attachments/assets/1bc73dd5-e72d-4bf3-8c24-630ca1fd073b" />

Tambien hay que modificar este archivo para que no permita la entrada de root , porque en esta distribución esta configuración se sobrepone al otro:


<img width="1021" height="1077" alt="Image" src="https://github.com/user-attachments/assets/0eaafd63-fc73-4c92-858f-3889c2da3853" />

Y al reinicar el servicio vemos que esta todo correcto por ahora :


<img width="1300" height="416" alt="Image" src="https://github.com/user-attachments/assets/f31c6533-3e30-4b54-838d-2260f570d537" />

   <h1> ◦ Tarea 2.3: Demostrar el uso de SSH para una sesión de trabajo remota y el uso de la opción -C (compresión) en enlaces lentos</h1>


Ahora pondremos aprueba si hemos realizado correctamente la configuración e instalacion de ssh y sus dependencias de seguridad.

<img width="1015" height="1080" alt="Image" src="https://github.com/user-attachments/assets/c1be2ed7-0972-473f-8e62-235fb0766a74" />



Y se ve como desde el cliente hemos accedido al Servidor .

Tambien si intentamos acceder desde otra IP da este resultado:


<img width="635" height="138" alt="Image" src="https://github.com/user-attachments/assets/d9d75462-5862-412c-b3f7-a1c073ada011" />



Y aunque la maquina permitida quiera acceder con root ,  no le dejara :

<img width="1023" height="1080" alt="Image" src="https://github.com/user-attachments/assets/1d692f1b-de9c-4503-b7e9-2269c040f677" />

Salta un error de permiso denegado



3. Especialista en GUI Remota (VNC/X11): Lidera la instalación de servicios de interfaz gráfica remota (e.g., VNC o retransmisión X11), y las pruebas de acceso heterogéneo (CE g). 

<p>Para poder llevar a cabo esta parte del proyecto, necesitamos instalar un servicio de administración remota, en este caso utilizaremos tightvncserver para poder realizar las tareas propuestas.</p>

<p>Para poder installar este servicio es tan sencillo como ejecutar este comando, como administrador</p>

```
apt install tightvncserver
```
<p>Despues de su instalación, falta configurar el escritorio remoto, para ello con tightvncserver :1 crea un escritorio remoto y despues de insertar una contraseña y verificarla, creara el escritorio remoto.</p>

<p>Ademas, es necesario instalar diferentes servicios para que esto pueda funcionar, o al menos intentar que funcione, estos son: redmmina, xtinghtvncserver, entre todo lo instalado con anterioridad.</p>

<img width="509" height="229" alt="Image" src="https://github.com/user-attachments/assets/5e957a1f-7f6d-4a8f-92d5-bb24b8882c8f" />

De seguido tendria que poder dejar conectarte a otras maquinas , pero por algun error que desconozco no me deja entrar a la otra maquina , cuando la configuracion esta bien hecha , hay conexión y el ssh funciona correctamente. 

Hemos intentado por VNC , pero en estas maquinas virtuales en clase , como hemos visto anteriormente en otros compañeros , no deja la funcionon de hacer el tunel , lo hemos intentado con el ordenador de mi compañero : Hugo , pero tampoco dejaba , creemos que es culpa del firewall o alguna configuración de la red.


<img width="1026" height="748" alt="Image" src="https://github.com/user-attachments/assets/186488d3-77c0-4bfd-adfe-c1a0b94a8110" />


4. Experto en Seguridad y Redes: Se centra en la implementación de mecanismos de encriptación (CE h), la valoración de la importancia de la seguridad (SSH vs Telnet), y la configuración de acceso (CE b, g).  

SSH (Secure Shell) es el protocolo moderno y esencial para la administración remota ya que su principal ventaja es el cifrado, lo que significa que todas las credenciales y datos de la sesión se codifican antes de viajar por la red protegiéndolos de la intercepción por otro lado Telnet es un protocolo obsoleto que no ofrece ningún tipo de cifrado, enviando nombres de usuario y contraseñas en texto plano, lo que lo hace completamente inseguro e inaceptable para cualquier tarea administrativa seria debido al alto riesgo de robo de credenciales.





PBI 3: Gestión de Transferencia Segura y Cuentas (CE f, c, e):
   
Tarea 3.1: Crear cuentas de usuario específicas para el acceso remoto y documentar el proceso

He creado los usuarios de todos los miembros del grupo mediante la orden “sudo adduser (nombre de la persona) y para comprobar que todos se hayan creado correctamente, lo he verificado usando la orden “cat /etc/passwd”, donde esta ruta contiene la lista de todas las cuentas de usuario y de sistema registradas en la máquina.

 
<img width="561" height="752" alt="Image" src="https://github.com/user-attachments/assets/6ded693c-4f58-4f85-867b-4f55fc5fdb06" />

<img width="538" height="413" alt="Image" src="https://github.com/user-attachments/assets/d423b3ab-2f99-4ba9-aebc-cb7b1b39c38b" />


 Tarea 3.2: Usar herramientas de administración remota de transferencia segura (scp o sftp) para copiar archivos de configuración. 

Hemos utilizado SCP para copiar archivos como /etc/hosts de manera cifrada entre el servidor y tu equipo aprovechando la seguridad de SSH pidiendo una contraseña para autenticar la conexión lo que garantiza que los datos viajen protegidos y es esencial para administrar configuraciones remotas.

<img width="377" height="46" alt="Image" src="https://github.com/user-attachments/assets/406ddd5e-94fc-4f58-8ed5-891dcde69e12" />



Tarea 3.3: Utilizar comandos de gestión de servicios (service o /etc/init.d/) vía SSH para iniciar o detener un demonio (servicio en segundo plano) de prueba, como cupsd
  
Primero mediante ssh me he conectado al cliente de cada uno de mis compañeros
 
<img width="632" height="173" alt="Image" src="https://github.com/user-attachments/assets/312384ae-ece4-4bcf-a1de-ec767f32b57e" />



Luego he comprobado en cada uno de ellos via SSH, que puedo iniciar y detener servicios en segundo plano como CUPS

<img width="574" height="283" alt="Image" src="https://github.com/user-attachments/assets/19c9b0cc-a401-477a-a25d-54d414780677" />


ERRORES QUE ME HE ENCONTRADO
 
1.

<img width="305" height="38" alt="Image" src="https://github.com/user-attachments/assets/1472bccb-bce8-4283-959c-ac21d7f37a68" />

No había podido iniciar el servicio ya que no le había dado permisos de administrador a los usuarios. SOLUCIÓN : He vuelto al usuario administrador y les he dado permiso con sudo usermod -aG sudo (nombre de usuario que quiero darle permisos)


2.

<img width="370" height="67" alt="Image" src="https://github.com/user-attachments/assets/9d16b3f9-ee92-44cb-9d8b-7df3cdd269f8" />

 



No me dejaba iniciar el servicio. SOLUCIÓN: No tenía el servicio instalado, simplemente lo he instalado y ya me dejaba iniciarlo.




Ingeniero de Automatización/GitHub: Responsable de la gestión del tablero en GitHub Projects (moviendo issues, asegurando la transparencia y la trazabilidad del Sprint Backlog).  


He creado un repositorio para hacer el proyecto todos juntos donde poder añadir y modificar cada uno sus archivos , y tener una tabla de seguimiento donde poder ver el estado de las tareas , quien lo ha creado , cuando ha sido comiteado y demás.


Aqui un video donde respalda el template y como lo he realizado.

https://github.com/user-attachments/assets/8e59a020-3ccb-4d72-9d18-aae4ba8493df


Tambien he añadido labels donde poder poner el tipo de trabajo que estan efectuando , en este caso todos son de documentación .

<img width="820" height="874" alt="Image" src="https://github.com/user-attachments/assets/4eb4c182-c281-4a38-8236-7c22fd7455d6" />


Por otra parte si entrar en cada Issue puedes ver quien ha asignado la tarea , la prioridad que tiene , la estimacion de la fecha cual debe de cumplir y la fecha de inicio , tambien si una tarea esta hecha entre mas personas se podría añadir colaboradores:


<img width="273" height="833" alt="Image" src="https://github.com/user-attachments/assets/9dd67ed5-5a83-4bf3-86ac-6621598977bb" />



Si nos vamos a la tabla de antes , podemos ver que ahora esta algo actualizada , y ahora , marca las fechas de cuando se inició el proyecto y cuando se tiene que finalizar:


<img width="1907" height="941" alt="Image" src="https://github.com/user-attachments/assets/b97cc94a-697b-414b-923a-7814a7dbdb27" /> 


 
