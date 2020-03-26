# Walid el Harrak
## The Crazy Thing
## Index
1. [Exemple-15. Ldap-remot i phpldapadmin-local](#E15)  
  1.1 [Desplegar el servei LDAP](#E15-1)  
  1.2 [Desplegar el servei PHPLDAPADMIN](#E15-2)  
2. [Exemple-16. Ldap-local i phpldapadmin-remot](#E16)  
  2.1 [Engegar LDAP i PHPLDAPADMIN i que tinguin connectivitat](#E16-1)    
  2.2 [Ara cal accedir des del host de l’aula al port 8080 del PHPLDAPADMIN per visualitzar-lo](#E16-2)

<a name="E15"></a>
## 1. Exemple-15. Ldap-remot i phpldapadmin-local
### Exemple-15 Ldap-remot i phpldapadmin-local
- **Explicació:**
- Desplegem dins d’un container Docker (host-remot) en una AMI (host-destí) el servei ldap amb el firewall de la AMI només obrint el port 22. Localment al host de
l’aula (host-local) desplegem un container amb phpldapadmin. Aquest container ha de poder accedir a les dades ldap. des del host de l’aula volem poder visualitzar el phpldapadmin.

#### Desplegar el servei ldap
- En el host-remot AMI AWS EC2 engegar un container ldap sense fer map dels ports.
- En la ami cal obrir únicament el port 22
- També cal configurar el /etc/hosts de la AMI per poder accedir al container ldap per nom de host (preferentment).
- verificar que des del host de l’aula (host-local) podem fer consultes ldap.

#### Desplegar el servei phpldapadmin
- Engegar en el host de l’aula (host-local) un container docker amb el servei phpldapadmin fent map del seu port 8080 al host-local (o no).
- Crear el túnel directe ssh des del host de l’aula (host-local) al servei ldap (host-remot) connectant via SSH al host AMI (host-destí).
- Configurar el phpldapadmin per que trobi la base de dades ldap accedint al host de l’aula al port acabat de crear amb el túnel directe ssh.
- Ara ja podem visualitzar des del host de l’aula el servei phpldapadmin, accedint al port 8080 del container phpldapadmin o al port que hem fet map del host de l’aula (si és que ho hem fet).

<a name="E15-1"></a>
### 1.1 Desplegar el servei LDAP
- 1. Ens connectem a la consola AWS:

```
[isx48144165@walid ~]$ ssh -i ~/.ssh/MyFedora.pem fedora@8.112.174.63
```

- 2. Posem en funcionament el container (sense fer mapping):

```
[fedora@ip-172-31-23-137 ~]$ docker run --rm --name ldapserver -h ldapserver --net mynet -d isx48144165/ldapserver19:latest
```

- 3. Comprovem que només el port 22 está obert:

```
[fedora@ip-172-31-23-137 ~]$ nmap localhost
[...]
PORT   STATE SERVICE
22/tcp open  ssh
```

- 4. Afegim al /etc/hosts la ip del nostre container:

```
[fedora@ip-172-31-23-137 ~]$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.19.0.2	ldapserver
```

- 5. Localment ens connectem directament al LDAP del servidor AWS:

```
[isx48144165@walid ~]$ ssh -i ~/.ssh/MyFedora.pem -L 5000:ldapserver:389 fedora@8.112.174.63

[isx48144165@walid ~]$ ldapsearch -x -LLL -h localhost:50000 -b 'dc=edt,dc=org' dn
dn: dc=edt,dc=org
dn: ou=maquines,dc=edt,dc=org
dn: ou=clients,dc=edt,dc=org
dn: ou=productes,dc=edt,dc=org
dn: ou=usuaris,dc=edt,dc=org
dn: ou=grups,dc=edt,dc=org
dn: cn=1asix,ou=grups,dc=edt,dc=org
[...]
```

<a name="E15-2"></a>
### 1.2 Desplegar el servei PHPLDAPADMIN
- 1. Engeguem el servidor phpldapadmin de forma interactiva ja que haurem de fer alguns canvis:

```
[isx48144165@walid ~]$ docker run --name phpldapadmin -h phpldapadmin --net mynet -p 8080:8080 -it isx48144165/phpldapadmin /bin/bash
```

- 2. Creem el tunel directe per accedir al phpldapadmin:

```
[isx48144165@walid ~]$ ssh -i MyFedora.pem -L 172.19.0.1:50000:ldapserver:389 fedora@8.112.174.63
```

- 3. Configurem el servidor phpldapadmin correctament:

```
[root@phpldapadmin docker]# vi /etc/phpldapadmin/config.php
$servers->setValue('server','host','172.19.0.1');
$servers->setValue('server','port',5000);
$servers->setValue('server','base',array('dc=edt,dc=org'));

# (Encenem els servidors)
[root@phpldapadmin docker]# /sbin/php-fpm
[root@phpldapadmin docker]# /sbin/httpd
```

- 4. Finalment, provem a connectar-nos des d'un navegador:

```
http://localhost:8080/phpldapadmin/

# I per poder entrar:
user: cn=Manager,dc=edt,dc=org
password: *password*
```

<a name="E16"></a>
## 2. Exemple-16. Ldap-local i phpldapadmin-remot
### Exemple-16. Ldap-local i phpldapadmin-remot
- **Explicació:**
- Obrir localment un ldap al host. Engegar al AWS un container phpldapadmin que usa el ldap del host d el’aula. Visualitzar localment al host de l’aula el phpldapadmin del container de AWS EC2. Ahí ez nà.

#### Engegar ldap i phpldapadmin i que tinguin connectivitat:
- Engegar localment el servei ldap al host-local de l’aula.
- Obrir un túnel invers SSH en la AMI de AWS EC2 (host-destí) lligat al servei ldap del host-local de l’aula.
- Engegar el servei phpldapadmin en un container Docker dins de la màquina AMI. cal confiurar-lo perquè connecti al servidor ldap indicant-li la ip de la AMI i el port obert per el túnel SSH.
- *nota* atenció al binding que fa ssh dels ports dels túnels SSH (per defecte són només al localhost).

#### Ara cal accedir des del host de l’aula al port 8080 del phpldapadmin per visualitzar-lo. Per fer-ho cal:
- En la AMI configutat el /etc/hosts per poder accedir per nom de host (per exemple php) al port apropiat del servei phpldapadmin.
- Establir un túnel directe del host de l’aula (host-local) al host-remot phpldapadmin passant pel host-destí (la AMI).
- Ara amb un navegador ja podem visualitzar localment des del host de l’aula el phpldapadmin connectant al pot directe acabat de crear.
- *nota* atenció al binding que fa ssh dels ports dels túnels SSH (per defecte són només al localhost).

<a name="E16-1"></a>
### 2.1 Engegar LDAP i PHPLDAPADMIN i que tinguin connectivitat
- 1. Engeguem localment un servidor ldap:

```
[isx48144165@walid ~]$ docker run --rm --name ldapserver -h ldapserver --net mynet -d isx48144165/ldapserver19:latest
```

- 2. Al /etc/hosts afegim la IP del docker:

```
172.19.0.2	ldapserver
```

- 3. Al fitxer /etc/ssh/sshd_config remot canviem l'opcio **GatewayPorts** a **yes**. D'aquesta manera els tunnels inversos poden fer bind en diferents interficies, ja que per defecte només permet localhost.

- 4. Engeguem servidor phpldapadmin al servidor amazon (remot):

```
[fedora@ip-172-31-23-137 ~]$  docker run --rm --name phpldapadmin -h phpldapadmin --net mynet -it isx48144165/phpldapadmin /bin/bash
```

- 5. Modifiquem, com hem fet anteriorment, la configuració de php:

```
[root@phpldapadmin docker]# vi /etc/phpldapadmin/config.php
$servers->setValue('server','host','172.19.0.1');
$servers->setValue('server','port',5000);
$servers->setValue('server','base',array('dc=edt,dc=org'))

# (I posem en marxa els servidors)
[root@phpldapadmin docker]# /sbin/php-fpm
[root@phpldapadmin docker]# /sbin/httpd
```

- 6. Creem un tunnel ssh invers cap al servidor amazon (remot):

```
[isx48144165@walid ~]$ ssh -i ~/.ssh/MyFedora.pem -R 172.19.0.1:5000:ldapserver:389 fedora@8.112.174.63
```

<a name="E16-2"></a>
### 2.2 Ara cal accedir des del host de l’aula al port 8080 del PHPLDAPADMIN per visualitzar-lo

- 1. Creeam un tunel directe al host local on obrim el port 8080 i ho redireix al servidor amazon (remot):
```
[isx48144165@walid ~]$ ssh -i MyFedora.pem -L 8080:172.19.0.2:80 fedora@8.112.174.63
```

- 2. Finalment, comprovem des del navegador si podem accedir-hi:

```
http://localhost:8080/phpldapadmin/

# I per poder entrar:
user: cn=Manager,dc=edt,dc=org
password: *password*
```
