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
 
