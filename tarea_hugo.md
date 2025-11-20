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
