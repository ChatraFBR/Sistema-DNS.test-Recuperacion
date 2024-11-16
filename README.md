## Autor
Fernando Bueno Rivera

### Introduccion

Esta documentación le proporciona la configuración básica y las configuraciones necesarias para configurar servidores DNS usando Bind9 con Vagrant.

## 1. Configurar el espacio de trabajo

### 1.1 Crear archivo Vagrantfile
Para iniciar la configuración, debes crear un archivo Vagrantfile usando el siguiente comando:
```ruby
    vagrant init
```

### 1.2 Editar archivo
El archivo Vagrantfile debe editarse para crear dos máquinas virtuales: una para el servidor DNS maestro (Tierra) y la otra para el servidor DNS esclavo (Venus). La siguiente configuración tiene dos disposiciones para cada una que instalan los paquetes necesarios, como bind9 y la configuración de archivos.

```ruby
Vagrant.configure("2") do |config|

  config.vm.box = "debian/bookworm64"

  config.vbguest.auto_update = false

  config.vm.define "earth" do |earth|
    earth.vm.network "private_network", ip: "192.168.57.103"

    earth.vm.hostname = "tierra.sistema.test"

    earth.vm.provision "shell", inline: <<-SHELL
        apt update -y
        apt-get install -y dnsutils bind9
    SHELL

    earth.vm.provision "shell", inline: <<-SHELL
      cp -v /vagrant/config/resolution/resolv.conf /etc/ 
      cp -v /vagrant/config/default/named /etc/default/named   
      cp -v /vagrant/config/options/named.conf.options /etc/bind/named.conf.options 
      cp -v /vagrant/config/earth/named.conf.local /etc/bind/named.conf.local
      mkdir -p /etc/bind/zones 
      cp -v /vagrant/config/earth/db.sistema.test /etc/bind/zones/db.sistema.test
      cp -v /vagrant/config/earth/db.192.168.57 /etc/bind/zones/db.192.168.57
      sudo systemctl restart bind9 
    SHELL
  end

  config.vm.define "venus" do |venus|
    venus.vm.network "private_network", ip: "192.168.57.102"

    venus.vm.hostname = "venus.sistema.test"

    venus.vm.provision "shell", inline: <<-SHELL
      apt update -y
      apt-get install -y dnsutils bind9
    SHELL

    
    venus.vm.provision "shell", inline: <<-SHELL
      cp -v /vagrant/config/resolution/resolv.conf /etc/
      cp -v /vagrant/config/default/named /etc/default/ 
      cp -v /vagrant/config/venus/named.conf.local /etc/bind/
      cp -v /vagrant/config/options/named.conf.options /etc/bind/
      mkdir -p /etc/bind/zones 
      touch /etc/bind/zones/db.sistema.test 
      touch /etc/bind/zones/db.192.168.57  
      sudo systemctl restart bind9 
    SHELL

  end

end

```

Para iniciar las máquinas virtuales, utilice el siguiente comando:
```bash
    vagrant up
```

### 1.3 Árbol de directorios

En el mismo directorio debe tener la siguiente organización:
```
   .
   ├── README.md
   ├── Vagrantfile
   ├── .vagrant
   │        └── ...
   └── config
           ├── options
           │       └── ...
           ├── defautl
           │       └── ...
           ├── earth
           │       └── ...
          └── venus
                  └── ...
```

## 2.  Configurar servidores

### 2.1 General configuration

#### 2.1.1 Escuchar solo IPv4

Modifique las opciones de inicio para escuchar solo en IPv4 agregando **`-4`**:
```bash
OPTIONS="-u bind -4"
```

#### 2.1.2 Opciones de configuración

##### Havilitar la validación dnssec

Edite las opciones de DNS en el archivo **named.conf.options** para habilitar la validación DNSSEC:
```bash
    sudo nano /etc/bind/named.conf.options
```

Modifíquelo de la siguiente manera:
```bash
    options {
            directory "/var/cache/bind";
            dnssec-validation yes; // dnssec-validation enable 
            listen-on { any; }; // Listening any ip
            listen-on-v6 { none; }; // Disabling IPv6 listening
    };
```

##### Configuración de ACL: recursiva a redes específicas

Para limitar las consultas recursivas a redes específicas, configure las ACL en el mismo archivo **named.conf.options**:
```bash
    sudo nano /etc/bind/named.conf.options
```

Lo añadiremos asi:
```bash
    acl "allow_networks" {
            127.0.0.0/8; // Allow queries from localhost
            192.168.57.0/24; // Allow queries from the VM network
    };
```

##### Servidor de reenvío para respuesta no autorizada

Edite las opciones de DNS en el archivo **named.conf.options**:

```bash
    sudo nano /etc/bind/named.conf.options
```
Y lo modificaremos asi:
```bash
    options {
            recursion yes;
            directory "/var/cache/bind";
            dnssec-validation yes;
            listen-on { any; };
            listen-on-v6 { none; };

            forwarders {
              208.67.222.222;  // Forward to OpenDNS
            };

            forward only;  // Indicate in case don't find resolution name, forward to another server
    };
```


#### 2.1.3 Configuración de la resolución

nos aseguraremos de que la configuración de resolución esté bien realizada
 
Editar  *(`resolv.conf`)*:
```bash
    sudo nano /etc/resolv.conf
```
sera *`nameserver`* el servidor DNS que resuelva los nombres
```bash
    sudo nameserver=192.168.57.103 // (IPv4 Master Server)
```


### 2.2 Servidor Maestro (earth)

Entramos en nuestro servidor maestro :
```bash
    vagrant ssh earth
```

#### 2.2.1 Configurar Local: Zona DNS

A continuación, definimos la zona DNS para tierra.sistema.test y su zona inversa en el archivo **named.conf.local**:
```bash
sudo nano /etc/bind/named.conf.local
```

Agregue la siguiente configuración de zona:
```bash
    zone "sistema.test" {
            type master;
            file "/etc/bind/zones/db.sistema.test";
};
      zone "57.168.192.in-addr.arpa" {
            type master;
            file "/etc/bind/zones/db.192.168.57";
};
```

Crea el directorio para los archivos de zona:
```bash
    sudo mkdir /etc/bind/zones
```

#### 2.2.2 Crear una base de datos de zona directa con alias y servidor de correo

Crear el archivo de zona DNS (`db.sistema.test`):
```bash
    sudo touch /etc/bind/zones/db.sistema.test
```
Luego, edite el archivo de base de datos de zona para (`sistema.test`):
```bash
    sudo nano /etc/bind/zones/db.sistema.test
```

En este archivo, agregue los registros DNS necesarios:
```bash
$TTL    604800
;
;
;dns an email
@       IN      SOA     tierra.sistema.test. admin.sistema.test. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                           7200 )       ; Negative Cache TTL 2 hours

;Name server record (NS)
@       IN      NS      tierra.sistema.test.

;Addres record (A)
venus   IN      A       192.168.57.102
@       IN      A       192.168.57.103
tierra  IN      A       192.168.57.103
marte   IN      A       192.168.57.104
mercurio    IN      A       192.168.57.101

; Alias (CNAME)
ns1     IN      CNAME   tierra.sistema.test.
ns2     IN      CNAME   venus.sistema.test.
mail    IN      CNAME   marte.sistema.test.

; Mail Exchange (MX)
@       IN      MX 10   marte ;Its unnecesary put .sistema.test.
```
##### ¿Qué significa "El servidor marte.sistema.test actuará como servidor de correo para el dominio sistema.test"?

Esta declaración significa que el servidor nombrado marte.sistema.testse encargará del procesamiento de correo electrónico para el dominio sistema.test . En otras palabras, cualquier correo electrónico enviado a direcciones como user@sistema.testserá administrado y procesado por el servidor marte.sistema.test.

Para configurar esto correctamente, debe agregar un registro MX (Mail Exchange) a su archivo de zona DNS. Un registro MX especifica el servidor de correo responsable de recibir los correos electrónicos de un dominio en particular.

#### 2.2.3 Crear base de datos de zona inversa

**Crear el archivo de zona DNS (`db.192.168.57`)**:
```bash
    sudo touch zones/db.192.168.57
```
**Luego, edite el archivo de base de datos de zona para (`57.168.192.in-addr.arpa`)**:
```bash
    sudo nano zones/zones/db.192.168.57
```

En este archivo, agregue los registros DNS necesarios:
```bash
$TTL    604800
@       IN      SOA     tierra.sistema.test. admin.sistema.test. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                           7200 )       ; Negative Cache TTL 2 hours

; Name Server (NS)
@       IN      NS      tierra.sistema.test.

; Pointer record (PTR)
101     IN      PTR     mercurio.sistema.test.
102     IN      PTR     venus.sistema.test.
103     IN      PTR     tierra.sistema.test.
104     IN      PTR     marte.sistema.test.
```

### 2.3 Servidor esclavo (Venus)

Entramos en nuestro servidor esclavo:
```bash
    vagrant ssh venus
```

#### 2.3.1 Configurar Local: Zona DNS

Entonces, vamos a crear dos zonas como servidor maestro, zona directa e inversa.
```bash
    sudo nano /etc/bind/named.conf.local
```

```bash
zone "sistema.test" {
        type slave; 
        file "/etc/bind/zones/db.sistema.test"; 
        masters { 192.168.57.103; };
};

zone "57.168.192.in-addr.arpa" {
        type slave; 
        file "/etc/bind/zones/db.192.168.57"; 
        masters { 192.168.57.103; }; 
};
```
#### 2.3.2 Configurar la base de datos DNS

Esta parte, solo crea una base de datos vacía con el comando **(`touch`)**:

```bash
    touch /etc/bind/zones/db.sistema.test
    touch /etc/bind/zones/db.192.168.57
```

## 3 Comprobar configuración

### 3.1 Lo comprobamos
```bash
    sudo named-checkconf
```
No devuelva la salida si todo está bien.

### 3.2 Comprobar configuración de zonas
Utilice este diseño para comprobar las configuraciones **`sudo named-checkzone yournamezone dbdirectory`**.

Zona directa
```bash
    sudo named-checkzone tierra.sistema.test /etc/bind/zones/db.sistema.test
```

Producción
```bash
    zone tierra.sistema.test/IN: loaded serial 2
    OK
```
Zona inversa
```bash
    sudo named-checkzone 57.168.192.in-addr.arpa /etc/bind/zones/db.192.162.57
```

Output
```bash
    zone tierra.sistema.test/IN: loaded serial 2
    OK
```

### 3.3 Verificar la resolución de nombres
El paso anterior verifica la sintaxis, así que vamos a verificar la resolución de DNS con nslookup.

Zona directa
```bash
    nslookup tierra.sistema.test
```

Producción
```bash
    Server:         192.168.57.103   
    Address:        192.168.57.103#53

    Name:   tierra.sistema.test      
    Address: 192.168.57.103
```

Zona inversa
```bash
    nslookup 192.168.57.102
```
Output
```bash
    102.57.168.192.in-addr.arpa     name = venus.sistema.test.
```

### 3.3 Verificar nombre con script

Con ese script puedes comprobar toda la configuración en una sola línea de comando.
```shell
    #!/bin/bash -x
    #
    # USAGE: ./test.sh <nameserver-ip>
    #

    # Salir si algún comando falla
    set -euo pipefail

    function resolver () {
        dig $nameserver +short $@
    }

    nameserver=@$1

    resolver mercurio.sistema.test
    resolver venus.sistema.test
    resolver tierra.sistema.test
    resolver marte.sistema.test

    resolver ns1.sistema.test
    resolver ns2.sistema.test

    resolver sistema.test mx

    resolver sistema.test ns

    resolver -x 192.168.57.101
    resolver -x 192.168.57.102
    resolver -x 192.168.57.103
    resolver -x 192.168.57.104
```
Para ejecutarlo ponga este comando así:

```bash
  ./nameScript IPv4DNSServer
```

### 4 Guardar configuración
#### Guardar archivos:

Al finalizar la configuración de su entorno Maestro/Esclavo, guarde la configuración de todos los archivos en su computadora invitada para establecer la disposición en VagrantFile .

##### General
```bash
    cp /etc/resolv.conf /vagrant/config/resolution/
    cp /etc/default/named /vagrant/config/default/
    cp /etc/bind/named.conf.options /vagrant/config/options/
```

##### Archivos Maestros
```bash
    cp /etc/bind/named.conf.local /vagrant/config/earth/
    cp /etc/bind/zones/db.sistema.test /vagrant/config/earth/
    cp /etc/bind/zones/db.192.168.57 /vagrant/config/earth/
```

##### Archivos esclavos
```bash
    cp /etc/bind/named.conf.local /vagrant/config/venus/
    cp /etc/bind/zones/db.sistema.test /vagrant/config/venus/
    cp /etc/bind/zones/db.192.168.57 /vagrant/config/venus/
```

#### La provisión de VagrantFile será así:

Asegúrese de crear el directorio necesario (como el directorio de zonas) y reinicie bind9 .

##### Maestro
```bash
earth.vm.provision "shell", inline: <<-SHELL
  # Here put file provision
  cp -v /vagrant/config/resolution/resolv.conf /etc/ 
  cp -v /vagrant/config/default/named /etc/default/named  
  cp -v /vagrant/config/options/named.conf.options /etc/bind/named.conf.options 
  cp -v /vagrant/config/earth/named.conf.local /etc/bind/named.conf.local 
  mkdir -p /etc/bind/zones 
  cp -v /vagrant/config/earth/db.sistema.test /etc/bind/zones/db.sistema.test 
  cp -v /vagrant/config/earth/db.192.168.57 /etc/bind/zones/db.192.168.57 
  sudo systemctl restart bind9 
SHELL
```

##### Esclavo
```bash
venus.vm.provision "shell", inline: <<-SHELL
  #Here put file provision
  cp -v /vagrant/config/resolution/resolv.conf /etc/ 
  cp -v /vagrant/config/default/named /etc/default/
  cp -v /vagrant/config/options/named.conf.options /etc/bind/  
  cp -v /vagrant/config/venus/named.conf.local /etc/bind/ 
  mkdir -p /etc/bind/zones 
  touch /etc/bind/zones/db.sistema.test 
  touch /etc/bind/zones/db.192.168.57  
  sudo systemctl restart bind9 
SHELL
```





