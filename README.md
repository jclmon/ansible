# ANSIBLE

## Instalación
```
$ apt-get update 
$ apt-get install software-properties-common python-software-properties
$ apt-add-repository ppa:ansible/ansible 
$ apt-get update 
$ apt-get install ansible
```
Añadimos las direcciones de nuestras máquinas Vagrant IP al fichero /etc/ansible/hosts en la máquina controller:
---
```
cat /etc/ansible/hosts
192.168.33.11 # node-one 
192.168.33.12 # node-two 
```

### Primeros comandos
Documentación ansible http://docs.ansible.com
```
ansible all -a "hostname" -f
ansible all -a "hostname" -f 1
ansible all -a "free -m"
ansible all -a "df -h" 
ansible all -a "date"
```

## Facts
En la jerga Ansible, los facts son los detalles de un determinado servidor o grupo de servidores. 
Es posible obtener una lista exhaustiva de los facts haciendo uso del módulo setup: 
```
$ ansible all -m setup 
```

## MÓDULOS 

### Módulo APT
Instalación de NTP (nombre y estado en el que debe estar)
```
$ ansible all -m apt -a "name=ntp state=present" -u root
$ ansible all -m apt -a "name=ntp state=absent" -u root
$ ansible all -a "service ntp status"
```

### Módulo SERVICIOS
Activación del servicio NTP
```
$ ansible all -s -m service -a "name=ntp 
state=started enabled=yes" 
```

### Módulo USUARIOS Y GRUPOS
Gestión de usuarios y grupos
Los módulos de Ansible hacen muy sencilla la gestión de usuarios y grupos. Por ejemplo, si quisiéramos añadir un grupo admin:
```
$ ansible all -m group -a "name=admin state=present" -u root
$ ansible all -m user -a "name=jc group=admin createhome=yes" -u root
```

### Módulo ARCHIVOS Y DIRECTORIOS
Si queremos obtener información de un archivo o si queremos obtener información sobre permisos o diferentes propiedades de un archivo, tenemos que utilizar el módulo stat: 
```
$ ansible all -m stat -a "path=/etc/hosts"
```
Copiar ficheros, el parámetro src puede ser un archivo o un directorio completo. Si incluimos una barra ‘/’ al final, 
solo los contenidos del directorio se copiaran al destino. 
```
$ ansible all -m copy -a "src=/etc/hosts dest=/tmp/hosts" 
```
Descargar un archivo de los servidores, el módulo fetch funciona igual que el copy pero a la inversa. 
Si antes copiábamos nuestro archivo hosts a las máquinas, ahora nos descargamos los suyos:
```
$ ansible all -m fetch -a "src=/etc/hosts dest=/tmp" 
```
Crear un archivo
```
$ ansible all -m file -a "path=/tmp/emptyfile.txt mode=644 state=touch" -u root
```
Eliminar archivos y directorios, para eliminar un archivo o directorio simplemente hay que establecer su estado a absent.
```
$ ansible all -m file -a "dest=/tmp/file state=absent" -u root
```
### Módulo CRON
Las tareas periódicas que se ejecutan mediante un cron se gestionan mediante el crontab del sistema. 
Lo normal en estos casos es ejecutar crontab -e para editar los crones, pero con el módulo de crones de Ansible, 
la gestión de los mismos es muy sencilla.

p.e. ejecutar un script todos los días a las 4 de la madrugada:
```
$ ansible all -u root -m cron -a "name='my-cron' hour=5 job='/script.sh'"
$ ansible all -u root -m cron -a "name='my-cron' hour=5 job='/script.sh' state=absent"
```

## PLAYBOOKS Y YAML
Los playbooks nos proporcionan una manera totalmente diferente de utilizar Ansible. Los comandos ad-hoc hacen de Ansible una herramienta muy potente, pero los playbooks la convierten en “la herramienta”. Mientras que podemos ejecutar comandos ad-hoc con Ansible, los playbooks podemos tenerlos en un control de versiones como Git. Además, pueden ejecutar tareas tal y como nosotros queramos haciendo uso de los inventarios, tags, roles, etc. 

Una de las cosas que más atrae a los DevOps a Ansible es que es muy sencillo convertir shell scripts en playbooks.

### YAML
Ansible utiliza YAML porque para los humanos es más fácil de leer que otros formatos como XML o JSON.
En Ansible, muchos archivos YAML comienzan con una lista. Cada item de la lista contiene una clave y/o valor, comúnmente llamado “hash” o “diccionario”. 
Antes de comenzar a escribir playbooks, debemos saber cómo declarar una de estas listas.

#### CONCEPTOS BÁSICOS DE YAML
Otra de las cosas importantes de YAML es que todos los archivos empiezan por guiones (---) y acaban con puntos (...).
Esto es opcional, pero recomendable para seguir con el estándar.
Todos los elementos de una lista son líneas que comienzan en el mismo nivel de indentación y cuyos dos primeros caracteres son un guión y un espacio “- “.
En YAML los comentarios comienzan con una almohadilla #.

### PLAYBOOKS
Un playbook se compone de varios plays. El objetivo de un play es mapear un grupo de hosts con unos determinados roles, 
los cuales son representados por tareas (tasks). En su expresión más básica, una tarea no es más que la llamada a un determinado módulo de Ansible.
Es posible orquestar muchas máquinas al mismo tiempo con los plays de Ansible gracias a la agrupación de los nodos en los inventarios de Ansible.
Con estos comandos podríamos instalar un servidor Apache en cada nodo:
```
$ sudo apt-get install apache2 
$ sudo service apache2 start 
```
Sería ...
apache-playbook.yml
---
```
	---
	- hosts: all
	  remote_user: root
	  tasks:
		- name: Install Apache2 package
		  apt: name=apache2 state=present
	...
```
Y se ejecuta
ansible-playbook apache-playbook.yml

Otro cambio en el playbook
---
```
---
- hosts: 192.168.33.11
  remote_user: root
  tasks:
    - name: Install Apache2 package
      apt: name=apache2 state=present
    - name: Ensure Apache2 service is running
      service: name=apache2 state=started enabled=yes
- hosts: 192.168.33.12
  remote_user: root
  tasks:
     - name: Uninstall Apache2
       apt: name=apache2 state=absent
     - name: Install haproxy load balanced
       apt: name=haproxy state=present
...
```

## HOSTS Y USUARIOS

Por cada play en un playbook debemos elegir cuáles son las máquinas objetivo para ejecutar las tareas.
- hosts: webservers 
remote_user: root 

El parámetro remote_user se refiere simplemente al usuario remoto para ejecutar las tareas.
Los usuarios remotos pueden definirse de un modo genérico para todas las tareas o también definirse por tareas individuales:
```
- hosts: webservers 
remote_user: root 
tasks: 
- name: remote user is mario 
copy: ... 
remote_user: mario 
```
Podemos ejecutar una tarea mediante escalado de privilegios haciendo uso del módulo become:
```
- hosts: webservers 
remote_user: yourname
become: yes 
become_user: www-data 
```
Además, es posible especificar el orden en el que se ejecutan las tareas en los hosts haciendo uso del módulo order:
```
- hosts: all 
order: sorted 
```

## LISTA DE TAREAS
Cada play contiene una lista de tareas, las cuales se ejecutan en orden y solo una al mismo tiempo. Es importante saber que el orden de las tareas es el mismo para todos los nodos.
Si falla una tarea, se puede volver a ejecutar, ya que los módulos deben ser idempotentes, es decir, que ejecutar un módulo varias veces debe tener el mismo resultado que si se ejecuta una vez.
Todas las tareas deben tener un name, el cual se incluye en la salida del comando al ejecutar el playbook. Su objetivo es que la persona que ejecute Ansible pueda leer con facilidad la ejecución del playbook.

Las tareas se declaran en el formato module: options. En un ejemplo anterior nos segurábamos de que el servicio apache2 se estuviese ejecutando:

```
--- 
hosts: all 
remote_user: root 
tasks: 
- name: ensure apache is running 
service: name=apache2 state=started enabled=yes 
``` 
Los módulos command y shell son los únicos comandos que no utilizan el formato clave=valor, sino que directamente se le pasa como parámetro el comando que deseamos ejecutar:
```
--- 
hosts: all 
remote_user: root 
tasks: 
- name: Run a command 
shell: /usr/bin/command 
```
## HANDLERS
Tal y como se ha mencionado anteriormente, los módulos deben ser idempotentes y pueden realizar cambios en los servidores donde se ejecuten. 
Ansible reconoce dichos cambios y tiene un sistema de eventos que se puede utilizar para gestionar el resultado de una acción.
Estas acciones de notificación son lanzadas al final de cada bloque de tareas en un play y solo se ejecutarán una vez incluso 
si se llaman desde diferentes tareas. Por ejemplo, cuando cambiemos un archivo de configuración podríamos reiniciar Apache, 
pero si Ansible detecta que lo vamos a reiniciar más de una vez, lo reinicia una única vez.
```
- name: configuration file 
template: src=template.j2 dest=/etc/file.conf 
notify: 
- restart php 
- restart apache 
```
handlers: 
---
```
- name: restart php 
service: name=php state=restarted 
- name: restart apache 
service: name=apache state=restarted 
```
Desde Ansible 2.2, los handlers pueden “escuchar” diferentes topics para evitar hacer llamadas a distintos handlers:
tasks: 
```
- name: restart everything 
command: echo "restart the web services" 
notify: "restart web services" 
handlers: 
- name: restart php 
service: name=php state=restarted 
listen: "restart web services" 
- name: restart apache 
service: name=apache state=restarted 
listen: "restart web services"
```

## OPCIONES
```
--inventory=PATH (-i PATH) 
$ansible-playbook -i development apache2-playbook.yml
```
Define un archivo de inventario situado en una determinada ruta con las IPs de los servidores. Si no especificamos este 
parámetro coge por defecto el inventario en /etc/ansible/hosts.
```
--verbose (-v) 
```
Ejecutar el playbook en modo verbose, es decir, mostrando la salida de todos los comandos.

Podemos pasarle la opción -vvvv para que nos de todos los detalles de la ejecución.
```
--extra-vars=VARS (-e VARS) 
```
Añadir variables adicionales a la ejecución del playbook en formato clave1=valor1,clave2=valor2
```
--forks=NUM (-f NUM) 
```
El número de forks para agrupar la ejecución de un playbook en diferentes servidores.
```
--connections=TYPE (-c TYPE) 
```
Se refiere al tipo de conexión que se va a utilizar. Por defecto es SSH, pero también podría ser local para ejecutar un playbook en el entorno local en el que se ejecuta Ansible.
```
--check 
```
Sirve para ejecutar un playbook en check mode, es decir, que todas las tareas definidas se comprobarán en todos los hosts, pero ninguna llegará a ejecutarse.

## HOSTS Y GRUPOS
Ansible ejecuta los playbooks en múltiples servidores al mismo tiempo dentro de una infraestructura.
Estos servidores se almacenan en un inventario, que por defecto es /etc/ansible/hosts, pero que nosotros podemos configurar con el parámetro -i.

El formato normal de un inventario es parecido al de un fichero .ini. Puede ser uno parecido a este:
```
mail.geekytheory.com 
[webserver] 
geekytheory.com 
[dbserver] 
database.geekytheory.com 
```

Las cabeceras entre corchetes son los nombres de grupo, los cuales se utilizan para clasificar sistemas y decidir cuáles vas a controlar en un determinado momento.

Es correcto poner varios sistemas en diferentes grupos, ya que una máquina puede ser al mismo tiempo un servidor web y un servidor de base de datos.

### HOSTS Y GRUPOS - PUERTOS
Si tienes sistemas que ejecutan SSH en un puerto no estándar, es posible especificar dicho puerto:
```
mail.geekytheory.com:2222 
[webserver] 
geekytheory.com:5658 
[dbserver] 
database.geekytheory.com:5301 
``` 

### HOSTS Y GRUPOS - PATRONES
En caso de que tengamos muchos hosts que siguen un mismo patrón, podemos hacer lo siguiente en lugar de ponerlos todos en la lista:
```
[webserver] 
Server[a:f]geekytheory.com 
[dbserver] 
database[1:50].geekytheory.com 
```
Además, podemos especificar estos parámetros con una notación clave=valor.
En versiones anteriores a Ansible 2.0, teníamos los parámetros ansible_ssh_user, ansible_ssh_host y ansible_ssh_port. Esto ha pasado a convertirse en ansible_user, ansible_host y ansible_port.

localhost ansible_connection=local si el servidor hace de controlador de ansible
```
[webserver] 
localhost ansible_connection=local 
one.foo.com ansible_connection=ssh ansible_user=mario 
two.foo.com ansible_connection=ssh ansible_user=mario 
```
Esto es una manera rápida de hacerlo, pero la correcta sería declarar estas variables dentro de la carpeta hosts_vars, pero ya discutiremos eso más adelante.


### HOSTS Y VARIABLES
Tal y como hemos visto anteriormente, Ansible nos permite especificar ciertos parámetros (variables) en el inventario. Además de esas variables predefinidas, podemos declarar otras:
```
[webserver] 
one.foo.com var1=foo var2=bar day=sun night=moon 
two.foo.com var1=foo var2=bar day=sun night=moon 
```

## GRUPOS DE VARIABLES
Además de por cada host, las variables pueden asignarse a un grupo completo con [grupo:vars]:
```
[webserver] 
one.foo.com 
two.foo.com 
[webserver:vars] 
var1=foo var2=bar day=sun night=moon 
 ```
## GRUPOS DE GRUPOS Y GRUPOS DE VARIABLES
Es posible hacer grupos de grupos utilizando el sufijo :children. También podemos seguir usando el sufijo :vars.
```
[geekytheory-web] 
www1.geekytheory.com 
www2.geekytheory.com 
[geekytheory-web:vars] 
ansible_user=geekytheory_web_user 
[geekytheory-db] 
database[01:03].geekytheory.com 
[geekytheory-log] 
log.geekytheory.com 
[geekytheory-backup] 
backup.geekytheory.com 
[geekytheory-nodejs] 
mad.geekytheory.com 
bcn.geekytheory.com 
nyc.geekytheory.com 
lon.geekytheory.com 
[geekytheory-nodejs:vars] 
ansible_ssh_user=geekytheory_node_user 
somevariable=itsvalue 

[ubuntu:children] 
geekytheory-web 
geekytheory-db 
geekytheory-nodejs 
[centos:children] 
geekytheory-log 
geekytheory-backup 
```
A pesar de que podemos definir variables en el inventario, es preferible almacenarlas en los archivos de configuración de cada host (lo veremos más adelante).

Los child groups tienen dos propiedades que merece la pena mencionar:

    Cada host miembro de un grupo hijo es miembro del grupo padre automáticamente.
    Las variables de un grupo hijo sobreescriben las de un grupo padre. 

Ejemplo de playbook para orquestar los servidores:
---
```
--- 
# Configuración común a todos los servidores. 
- hosts: all 
sudo: true 
roles: 
- security 
- logging 
- firewall 
- zabbix-agent 
# Configuración de los servidores web. 
- hosts: geekytheory-web 
roles: 
- nginx 
- php 
- geekytheory-web 
# Configuración de los servidores de base de datos. 
- hosts: geekytheory-db 
roles: 
- mysql 
- mongodb 
# Configuración del servidor de logs/monitorización. 
- hosts: geekytheory-log 
roles: 
- loggly 
- zabbix-server 
# Configuración del servidor de backup. 
- hosts: geekytheory-backup 
roles: 
- backup 
# Configuración del servidor NodeJS. 
- hosts: geekytheory-nodejs 
roles: 
- nodejs 
```
En los inventarios de Ansible existen dos grupos por defecto: all y ungrouped. all contiene a todos los hosts y ungrouped contiene a los hosts que no están asociados a ningún grupo.

## ORGANIZACIÓN DE VARIABLES
Una buena práctica en Ansible es la de no almacenar las variables en el fichero principal de inventario.
Es posible separar las variables para determinados grupos o para determinar.

Aunque es posible escribir un playbook en un archivo muy largo, lo habitual es tratar de reutilizar partes del proyecto Ansible y tener todo organizado. En su nivel más básico, incluir archivos con tareas nos permite dividir grandes archivos de configuración en otros más pequeños.

Los roles en Ansible se utilizan con la idea de que incluir archivos con tareas y combinarlos pueden formar un proyecto limpio y con partes reutilizables. Comenzaremos explicando lo que son los includes antes de aprender a controlar los roles.

# ROLES E INCLUDES
Ya hemos visto en capítulos anteriores cómo instalar un servidor Apache y asegurarnos de que se esté ejecutando en varios servidores. ¿Y si quisiéramos reutilizar esas instrucciones?

## INCLUDES DE TAREAS
Este es el playbook original:
```
- hosts: all 
remote_user: root 
tasks: 
- name: Ensure Apache is at the latest version 
apt: name=apache2 state=latest 
- name: ensure apache is running 
service: name=apache2 state=started enabled=yes 
 ```
Podríamos dejarlo así:
```
- hosts: all 
remote_user: root 
tasks: 
- include: apache-playbook.yml 
``` 
El fichero apache-playbook.yml debería quedar así:
---
```
--- 
- name: Ensure Apache is at the latest version 
apt: name=apache2 state=latest 
- name: ensure apache is running 
service: name=apache2 state=started enabled=yes 
 ```

Y podríamos reutilizarlo desde cualquier parte del proyecto.

## INCLUDES DE HANDLERS
Los handlers pueden incluirse de igual manera que las tareas, pero en la sección correspondiente. En capítulos anteriores teníamos este ejemplo:
```
handlers: 
- name: restart php 
service: name=php state=restarted 
- name: restart apache 
service: name=apache state=restarted 
 ```
Para poder reutilizar los handlers, podríamos sacarlos a un fichero externo, como includedhandlers.yml, cuyo contenido sería:
```
 - name: restart php 
service: name=php state=restarted 
- name: restart apache 
service: name=apache state=restarted 
 ```
Y en la sección de handlers tendríamos que hacer el include:
```
handlers:
    include: included-handlers.yml 
```
## INCLUDES DE PLAYBOOKS

Un playbook puede ser incluido en otro playbook utilizando la misma sintaxis de include. Por ejemplo, si tenemos un playbook que configura los servidores web y otro que configura los servidores de base de datos, podríamos hacer un include de ambos playbooks en uno solo:
```
    include: webserver-playbook.yml
    include: database-playbook.yml 
```
Un playbook puede ser incluido en otro playbook utilizando la misma sintaxis de include. Por ejemplo, si tenemos un playbook que configura los servidores web y otro que configura los servidores de base de datos, podríamos hacer un include de ambos playbooks en uno solo:
```
    include: webserver-playbook.yml
    include: database-playbook.yml 
```

## INCLUDES DINÁMICOS
A partir de Ansible 2.0 los includes pueden ser dinámicos haciendo uso de bucles:
```
--- 
hosts: all 
remote_user: root 
tasks: 
- include: foo.yml param={{ item }} 
with_items: 
- 1 
- 2 
- 3 
 ```
foo.yml: 
---
```
--- 
name: "Echoing params" 
command: echo "{{ param }}" 
 ```
 
### EJEMPLO COMPLETO CON INCLUDES
```
- hosts: all 
pre_tasks: 
- name: Update apt cache if needed. 
apt: update_cache=yes cache_valid_time=3600 
handlers: 
- include: handlers/handlers.yml 
tasks: 
- include: tasks/common.yml 
- include: tasks/apache.yml 
- include: tasks/php.yml 
- include: tasks/mysql.yml 
```

Separar las tareas en diferentes archivos quiere decir que tendremos más archivos que gestionar, pero hará que nuestro playbook sea más mantenible. Es mucho más fácil mantener pequeños grupos de tareas relacionadas entre sí que un playbook excesivamente largo. El uso de tags también facilita la organización en un proyecto.

## ROLES
Ahora que ya sabemos lo que es una tarea, un handler o un include, sabemos que un proyecto en Ansible puede ser organizado para que sea mantenible. Si ya sabemos todo esto, ¿para qué sirven los roles?

Cuando nuestra aplicación crece y hacemos includes, podemos llegar a transformar la aplicación en algo ilegible porque tendríamos un include dentro de otro infinitamente.

Aquí surgen los roles, que son “paquetes” de configuraciones bien organizados que pueden reutilizarse en cualquier host.

En un role podemos detectar si estamos en un sistema operativo u otro, podemos crear variables, templates, archivos, etc. Todo ello de una manera estructurada.

Un role no es más que muchos includes organizados y estructurados.

Esta podría ser la estructura de un proyecto:
```
webservers.yml 
roles/ 
	- common/ 
	- files/ 
	- templates/ 
	- tasks/ 
	- handlers/ 
	- vars/ 
	- defaults/ 
	- meta/ 
 
```
### ROLES - ESTRUCTURA
En un playbook podríamos utilizarlo de la siguiente manera:
```
    hosts: webservers
    roles: common 
```
El comportamiento de un role es:
---
    Si roles/x/tasks/main.yml existe, las tareas dentro de dicho archivo se añadirán al play.
    Si roles/x/handlers/main.yml existe, los handlers dentro de dicho archivo se añadirán al play.
    Si roles/x/vars/main.yml existe, las variables dentro de dicho archivo se añadirán al play.
    Si roles/x/defaults/main.yml existe, las variables dentro de dicho archivo se añadirán al play.
    Si roles/x/meta/main.yml existe, las dependencias listadas en dicho fichero se añadirán a la lista de roles.
    Las carpetas roles/x/files y roles/x/templates se utilizan para guardar ficheros y plantillas, que por lo general son de configuración.

Si algún directorio no existe, simplemente se ignora.

### ROLES PARAMETRIZADOS
De igual manera que los includes, los roles también pueden recibir parámetros añadiendo variables para ser reutilizados. Por ejemplo:
```
- hosts: webservers 
roles: 
- common 
- { role: custom_role, foo: ‘1’, bar: ‘1’ } 
- { role: custom_role, foo: ‘2’, bar: ‘2’ } 
``` 
Es muy útil añadir tags a los roles para que sea más fácil ejecutarlos a nuestro gusto:
---
```
- hosts: webservers 
roles: 
- { role: common, tags: “common” } 
- { role: custom_role, tags: “custom” } 
``` 
Con esto asignamos tags a todas las tareas de un role, es decir, que las sobreescribimos.

### ROLES - PRE_TASKS y POST_TASKS
Si queremos que determinadas tareas se ejecuten antes y después de que los roles sean aplicados, debemos utilizar las pre_tasks y post_tasks:
```
- hosts: webservers 
pre_tasks: 
- shell: echo “Hello World” 
roles: 
- { role: common, tags: “common” } 
tasks: 
- shell: echo “Running a task” 
post_tasks: 
- shell: echo “Bye!” 
 ```

### ROLES - VARIABLES POR DEFECTO
Las variables por defecto de un role se añaden a default/main.yml dentro del directorio de nuestro role.
Estas variables tienen la prioridad más baja de entre todas las variables disponibles y pueden ser sobreescritas fácilmente por otra variable, incluidas las de los inventarios.

### ROLES - DEPENDENCIAS
Las dependencias nos permiten ejecutar roles dentro de roles. Se almacenan dentro del directorio meta/main.yml.
Este archivo debe incluir una lista de dependencias de roles y parámetros a ejecutar. dependencies:
```
- { role: '/path/to/roles/foo', bar: 50 } 
- { role: apache, port: 80 } 
``` 
Las dependencias de un role se ejecutan antes que el propio role. Si un role es una dependencia de otro no se ejecutará. Si queremos, podemos indicarle que se ejecute de nuevo haciendo uso de la directiva allow_duplicates.
```
allow_dupliates: yes #Por defecto es ‘no’ 
dependencies: 
- { role: '/path/to/roles/foo', bar: 50 } 
- { role: apache, port: 80 } 
``` 
