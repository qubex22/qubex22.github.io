---
title: Configuración VyOS para Movistar
tags: vyos movistar network iptv
---

# Configuración VyOS para Movistar +Bonus Track IPTV
> Index
> -pppoe (pppoe0 adjust-mss '1452')
> -snat masquerade
> -firewall
> -dnat port forward y hairpin
> 
> IPTV
> -interfaz ip publica (como se obtiene)
> -source nat masquerade
> -dhcp
> -igmp-proxy
> -solo un deco dnat para VOD, +2 rtsp-linux

---
## ¿Qué es VyOS?
https://vyos.io
VyOS es un router open source basado en debian. Es la evolución del conocido Vyatta, el cual también es la base de EdgeOS de ubiquiti, lo que le hace una plataforma muy madura y útil.
Tiene una comunidad muy activa y sus posibilidades son infinitas, además la configuración es muy intuitiva a pesar de ser por linea de comandos. 
Yo personalmente lo utilizo desde hace un par de años después de haber estado mucho tiempo con un router debian y nunca he tenido ningún problema, y me ha permitido hacer todo lo que se me ha ido ocurriendo.
Lo tengo montado en un minipc fanless de 4 interfaces, dejo el enlace [por aquí](https://es.aliexpress.com/item/1005001794025490.html?spm=a2g0s.9042311.0.0.274263c06IZZI4), pero se puede instalar en cualquier dispositivo x86 o virtualizado. 

## Descarga
Se puede obtener gratis de varias maneras.
- Descargar la iso rolling release: https://vyos.net/get/nightly-builds/
- Crear la iso manualmente: https://docs.vyos.io/en/latest/contributing/build-vyos.html
- Suscripción de colaborador/academica/gobierno/sin animo de lucro: https://vyos.net/get/contributor-subscriptions/

La instalación es sencilla simplemente crea un usb bootable, inicia y ejecuta el comando `install image`

## Introducción
VyOS tiene 2 modos de línea de comandos, que si alguna vez has tocado un router cisco o similares verás que es muy parecido:
1. Modo operacional `$` en el que se pueden ejecutar comandos relacionados con las tareas del sistema, estadisticas, estado... todo lo que no necesite permisos elevados.
2. Modo de configuración `#` donde se realizan las configuraciones del sistema. Para entrar a este modo basta con escribir configure.

La configuración se guarda en un archivo ubicado en ```/config/config.boot``` y se guarda cada vez que ejecutamos el comando ```save``` en el modo de configuración.
- Para revisar esta configuración utilizaremos ```show configuration``` para una salida jerarquizada:
```
vyos@vyos:~$ show configuration
interfaces {
    ethernet eth0 {
        address dhcp
        hw-id 00:53:00:00:aa:01
    }
    loopback lo {
    }
}
service {
    ssh {
        port 22
    }
}
system {
    config-management {
        commit-revisions 20
    }
    console {
        device ttyS0 {
            speed 9600
        }
    }
    login {
        user vyos {
            authentication {
                encrypted-password ****************
            }
            level admin
        }
    }
    ntp {
        server 0.pool.ntp.org {
        }
        server 1.pool.ntp.org {
        }
        server 2.pool.ntp.org {
        }
    }
    syslog {
        global {
            facility all {
                level notice
            }
            facility protocols {
                level debug
            }
        }
    }
}
```
- `show conf commands` para ver todos los comandos necesarios (útil para backups):
```
vyos@vyos:~$ show configuration commands
set interfaces ethernet eth0 address 'dhcp'
set interfaces ethernet eth0 hw-id '00:53:dd:44:3b:0f'
set interfaces loopback 'lo'
set service ssh port '22'
set system config-management commit-revisions '20'
set system console device ttyS0 speed '9600'
set system login user vyos authentication encrypted-password '$6$Vt68...QzF0'
set system login user vyos level 'admin'
set system ntp server '0.pool.ntp.org'
set system ntp server '1.pool.ntp.org'
set system ntp server '2.pool.ntp.org'
set system syslog global facility all level 'notice'
set system syslog global facility protocols level 'debug'
```
## Diagrama Movistar FTTH
Primero que nada se da por hecho que se ha sustituido el HGU de movistar por ONT+Router o HGU en modo bridge.
Este es el esquema de red con el que vamos a trabajar
![](https://camo.githubusercontent.com/39031ce095d3b7ea77b33d6ca50055c4105ce793ecf4ba28f264b69bb0a036a6/68747470733a2f2f692e706f7374696d672e63632f357464626a3178782f65737175656d617265642d64726177696f2e706e67)

- Mi ONT es el modelo [HG8310M de Huawei](https://es.aliexpress.com/item/4000747557354.html?spm=a2g0s.9042311.0.0.274263c0BAHXDX), **IMPORTANTE**, tiene que tener el conector verde **SC APC**, el azul no es compatible con nuestra instalación de movistar. También he probado con una **Alcatel g-010g-p** que tenía por casa, es la que suele poner vodafone, y funciona perfectamente. Avisaros de que si tenéis la **ONT de ubiquiti** (ufiber nano/loco), la TV no os va a funcionar, ya que no permite multicast. [Aquí](https://community.ui.com/questions/Adding-IGMP-multicast-on-UFiber-Nano-G/c0e37bc7-e14b-4a79-8ad1-90c6e168d0c7 "Aquí") os dejo un enlace de su foro si teneis curiosidad
- Movistar hace uso de **VLAN** en su infraestructura, esto les permite mandar por el mismo canal varios servicios (TV, internet, Voz ip). Sin embargo cada uno de estos servicios utiliza diferentes sistemas:
	1. Los datos de **internet** viajan a traves del protocolo **PPPoE**. Un protocolo de encapsulación bastante maduro que, aunque no es muy eficaz a altas velocidades y luego veremos por qué, su configuración es sencilla para nosotros y la implementacion de la infraestructura poco costosa para Movistar.
	2. En cuanto al servicio de **TV**, Movistar nos asigna una direccion **ip fija**, la cual, se telecarga al router que nos instalan en el primer encendido. Por ello que sea necesario apuntarla, como expliqué en la entrada anterior. Además Movistar utiliza el protocolo **RIP de rutas dinámicas**, con el cual nos mandará las rutas necesarias para la visualizacion de la TV.
	3. Por último movistar asigna ip dinámicas por **DHCP** para el servicio de Telefonía **Voz IP**

- Creo que un **switch PoE** es esencial si quieres poner un punto de acceso. En mi caso tengo un `NetGear GS108PE`, es gestionado, pero no sería necesario si no vas a tener varias vlan en casa (Wifi invitados, IOT, Alexa, etc...)
- Para el **punto de acceso** hay bastante mercado, ya depende del uso que le des al Wifi. Yo recomendaría calidad/precio un `Tplink EAP 225` o un `Ubiquiti AP AC Lite`

## Configuración para Movistar FTTH

### 1. Interfaces:
Primero configuraremos las interfaces y las vlan necesarias (en el **modo configuración #** siempre):
- eth0 será la interfaz wan física, donde configuraremos las vlan. Si tenemos tv necesitaremos la ip que nos asigna movistar 10.1xx.xxx.xxx/10, hay tutoriales por ahi de como conseguirla
- eth0.2 será la interfaz wan TV
- eth0.6 será la interfaz wan pppoe
- eth1 será la interfaz local de internet
- eth2 será la interfaz para TV local
- En el caso de que nuestro dispositivo no tenga bocas suficientes podemos utilizar vlan, es decir podríamos juntar las dos redes locales en la misma boca y asignarle vlan distintas.
- Por último configuramos el acceso al pppoe y ciertos parámetros todos ellos necesarios.
```
set interfaces ethernet eth0 vif 2 address '10.1xx.xxx.xxx/10'
set interfaces ethernet eth0 vif 2 description 'Movistar TV'
set interfaces ethernet eth0 vif 2 mtu '1500'
set interfaces ethernet eth0 vif 6 description 'Internet (PPPoE)'
set interfaces ethernet eth1 address '192.168.1.1/24'
set interfaces ethernet eth1 description 'Local'
set interfaces ethernet eth2 description 'Movistar TV'
set interfaces ethernet eth2 address '192.168.98.1/24'

set interfaces pppoe pppoe0 authentication password 'adslppp'
set interfaces pppoe pppoe0 authentication user 'adslppp@telefonicanetpa'
set interfaces pppoe pppoe0 connect-on-demand
set interfaces pppoe pppoe0 default-route 'auto'
set interfaces pppoe pppoe0 firewall in name 'WAN_IN'
set interfaces pppoe pppoe0 firewall local name 'WAN_LOCAL'
set interfaces pppoe pppoe0 firewall out name 'WAN_OUT'
set interfaces pppoe pppoe0 mtu '1492'
set interfaces pppoe pppoe0 source-interface 'eth0.6'
```

### 2. Source NAT
Necesario para funcione. Esto hace que todos los paquetes "salgan" por la wan con nuestra ip pública y cuando vuelvan "sepan" a qué dispositivo tienen que ir.

```
set nat source rule 1 outbound-interface 'pppoe0'
set nat source rule 1 protocol 'all'
set nat source rule 1 source
set nat source rule 1 translation address 'masquerade'
set nat source rule 2 log 'enable'
set nat source rule 2 outbound-interface 'eth0.2'
set nat source rule 2 protocol 'all'
set nat source rule 2 source
set nat source rule 2 translation address 'masquerade'
```

### 3. Firewall
- Parámetro necesario
```
set firewall options interface pppoe0 adjust-mss '1452'
```
- De fuera hacia dentro
```
set firewall name WAN_IN default-action 'drop'
set firewall name WAN_IN description 'WAN to internal'
set firewall name WAN_IN rule 20 action 'accept'
set firewall name WAN_IN rule 20 description 'Allow established/related'
set firewall name WAN_IN rule 20 state established 'enable'
set firewall name WAN_IN rule 20 state related 'enable'
set firewall name WAN_IN rule 30 action 'drop'
set firewall name WAN_IN rule 30 description 'Drop invalid state'
set firewall name WAN_IN rule 30 state invalid 'enable'
set firewall name WAN_IN rule 40 action 'drop'
set firewall name WAN_IN rule 40 description 'DROP ICMP'
set firewall name WAN_IN rule 40 protocol 'icmp'
```
- De fuera hacia el router
```
set firewall name WAN_LOCAL default-action 'drop'
set firewall name WAN_LOCAL description 'WAN to router'
```
- De dentro hacia fuera
```
set firewall name WAN_OUT default-action 'accept'
set firewall name WAN_OUT description 'from home to ext'
```

### 4. Port Forward
- En la parte de firewall creamos la regla para el puerto e ip de destino, esta por ejemplo es para un reverse proxy
```
set firewall name WAN_IN rule 51 action 'accept'
set firewall name WAN_IN rule 51 description 'reverse http'
set firewall name WAN_IN rule 51 destination address '192.168.1.250'
set firewall name WAN_IN rule 51 destination port '1880'
set firewall name WAN_IN rule 51 protocol 'tcp'
set firewall name WAN_IN rule 51 state new 'enable'
```
- En la parte de nat creamos una regla DNAT (Destination Nat)
```
set nat destination rule 10 description 'reverse http'
set nat destination rule 10 destination port '80'
set nat destination rule 10 inbound-interface 'pppoe0'
set nat destination rule 10 protocol 'tcp'
set nat destination rule 10 translation address '192.168.1.250'
set nat destination rule 10 translation port '1880'

```

TODO:
** HAIRPIN
**IPTV





























