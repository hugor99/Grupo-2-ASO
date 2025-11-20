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
