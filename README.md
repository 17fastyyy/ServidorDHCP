# ServidorDHCP

Práctica de instalación y configuración de un servidor DHCP con **Kea** en **Ubuntu Server**, con un cliente **Zorin OS** y análisis del tráfico con **Wireshark**.

- **Alumno:** Toni Correa
- **Número de lista (x):** 7 → red **192.169.7.0/24**

## Índice

1. [Esquema de la práctica](#1-esquema-de-la-práctica)
2. [Preparación de las máquinas en VirtualBox](#2-preparación-de-las-máquinas-en-virtualbox)
3. [Configuración de red del servidor](#3-configuración-de-red-del-servidor)
4. [Instalación de Kea y desactivación de DHCPv6 y DDNS](#4-instalación-de-kea-y-desactivación-de-dhcpv6-y-ddns)
5. [Configuración del archivo kea-dhcp4.conf](#5-configuración-del-archivo-kea-dhcp4conf)
6. [Comprobación de la sintaxis y arranque del servicio](#6-comprobación-de-la-sintaxis-y-arranque-del-servicio)
7. [Wireshark en el cliente](#7-wireshark-en-el-cliente)
8. [Captura de la negociación DHCP](#8-captura-de-la-negociación-dhcp)
9. [Análisis de los paquetes: broadcast y unicast](#9-análisis-de-los-paquetes-broadcast-y-unicast)
10. [Comprobación del cliente](#10-comprobación-del-cliente)
11. [Concesiones del servidor](#11-concesiones-del-servidor)
12. [Reserva de una IP para el cliente](#12-reserva-de-una-ip-para-el-cliente)
13. [Conclusiones](#13-conclusiones)

---

## 1. Esquema de la práctica

| Máquina | Interfaz | Modo en VirtualBox | Dirección |
|---|---|---|---|
| Ubuntu Server (`srv-smx01`) | `enp0s3` | NAT | DHCP de VirtualBox (acceso a Internet) |
| Ubuntu Server (`srv-smx01`) | `enp0s8` | Red interna `Internet` | **192.169.7.1/24** (estática) |
| Zorin (`fxsty-VirtualBox`) | `enp0s3` | NAT → Red interna `Internet` | Recibida por DHCP |

Parámetros que reparte el servidor DHCP:

| Parámetro | Valor |
|---|---|
| Subred | 192.169.7.0/24 |
| Pool | 192.169.7.10 – 192.169.7.50 |
| Puerta de enlace | 192.169.7.254 |
| DNS | 8.8.8.8 |
| Reserva | 192.169.7.55 para la MAC `08:00:27:a8:06:a5` (Zorin) |

---

## 2. Preparación de las máquinas en VirtualBox

El servidor necesita dos interfaces: la primera en **NAT**, que le da acceso a Internet, y la segunda en **Red interna**, que es por donde dará el servicio DHCP.

**Adaptador 1 del servidor en NAT:**

![Adaptador 1 del servidor en NAT](media/cap1.png)

**Adaptador 2 del servidor en Red interna**, con el nombre `Internet`. El cliente tendrá que usar exactamente el mismo nombre para estar en la misma red:

![Adaptador 2 del servidor en Red interna](media/cap2.png)

El **Zorin** empieza con su adaptador en **NAT**, para poder instalar Wireshark desde Internet. Más adelante se cambiará a Red interna:

![Adaptador del Zorin en NAT](media/cap3.png)

---

## 3. Configuración de red del servidor

Se configura la red con **netplan** (`/etc/netplan/50-cloud-init.yaml`):

- `enp0s3` (NAT) recibe la IP por DHCP.
- `enp0s8` (red interna) tiene la IP estática **192.169.7.1/24**, **sin puerta de enlace ni servidor DNS**, tal como pide el enunciado.

![Archivo de netplan](media/cap4.png)

Se aplican los cambios con `sudo netplan apply` y se comprueba con `ip a` que la interfaz `enp0s8` tiene la IP 192.169.7.1:

![ip a del servidor con la 192.169.7.1](media/cap5.png)

---

## 4. Instalación de Kea y desactivación de DHCPv6 y DDNS

Se instala Kea:

```bash
sudo apt update
sudo apt install kea
```

El paquete instala varios servicios: el servidor DHCPv4, el DHCPv6, el DDNS y el agente de control. En esta práctica solo se usa **DHCPv4**, así que hay que desactivar el **DHCPv6** y el **DDNS**.

En la presentación se indica que se haga en el archivo `keactrl.conf`, pero en Ubuntu **este archivo no existe**, porque los servicios de Kea se gestionan con **systemd**. Por eso se paran y desactivan con `systemctl`:

```bash
sudo systemctl disable --now kea-dhcp6-server
sudo systemctl disable --now kea-dhcp-ddns-server
```

Se comprueba que los dos servicios han quedado desactivados (`disabled`) y parados (`inactive`):

![DHCPv6 y DDNS desactivados](media/cap6.png)

---

## 5. Configuración del archivo kea-dhcp4.conf

Primero se renombra el archivo original para no perderlo y poder consultarlo como ejemplo. Después se crea uno nuevo:

```bash
cd /etc/kea
sudo mv kea-dhcp4.conf old-kea-dhcp4.conf
sudo nano kea-dhcp4.conf
```

Contenido del archivo nuevo:

![Archivo kea-dhcp4.conf](media/cap7.png)

Explicación de cada parte:

| Parámetro | Para qué sirve |
|---|---|
| `interfaces-config` | Interfaz por la que escucha el servidor: `enp0s8`, la de la red interna. |
| `valid-lifetime` | Duración de la concesión: 4000 segundos. |
| `renew-timer` | A los 1000 segundos el cliente intenta renovar la concesión con el mismo servidor. |
| `rebind-timer` | A los 2000 segundos, si el servidor no ha contestado al renew, el cliente pide la renovación a cualquier servidor. |
| `lease-database` | Las concesiones se guardan en un archivo (`memfile`) de forma persistente, en `/var/lib/kea/kea-leases4.csv`. |
| `subnet4` → `id` | Identificador de la subred. Las versiones nuevas de Kea lo piden; si no se pone, sale un aviso. |
| `subnet4` → `subnet` | La subred de trabajo: 192.169.7.0/24. |
| `pools` | Rango de direcciones que se reparten: de la .10 a la .50. |
| `option-data` → `routers` | Puerta de enlace que se envía al cliente: 192.169.7.254. |
| `option-data` → `domain-name-servers` | Servidor DNS que se envía al cliente: 8.8.8.8. |

Al ser un archivo **JSON** hay que fijarse bien en el formato: los bloques van entre `{ }`, las listas entre `[ ]`, los elementos se separan con comas y el último elemento de un bloque **no** lleva coma.

---

## 6. Comprobación de la sintaxis y arranque del servicio

Se comprueba la sintaxis del archivo:

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

![Comprobación de la sintaxis](media/cap8.png)

No aparece ningún `ERROR` (los `WARN` son avisos normales del multithreading). En la salida se ve que la configuración se ha leído bien: se ha añadido la subred **192.169.7.0/24** con los temporizadores t1=1000, t2=2000 y valid-lifetime=4000, y el servidor escucha en la interfaz **enp0s8**.

> **Problema encontrado:** la primera vez que se comprobó la sintaxis aparecía la subred 192.168.200.0/24 y la interfaz enp0s3. Era la configuración del archivo original, porque el archivo nuevo no se había guardado bien. Se solucionó borrando el `kea-dhcp4.conf` y creándolo otra vez.

Se reinicia el servicio y se comprueba su estado:

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

![Servicio kea-dhcp4-server en marcha](media/cap9.png)

El servicio está **active (running)** y se ejecuta con el archivo `/etc/kea/kea-dhcp4.conf`.

---

## 7. Wireshark en el cliente

Con el Zorin todavía en NAT se instala Wireshark:

```bash
sudo apt install wireshark
```

Durante la instalación pregunta si los usuarios sin privilegios pueden capturar paquetes. Se ha contestado **No**, que es la opción recomendada por seguridad, ya que Wireshark se abrirá con `sudo`.

![Wireshark instalado](media/cap10.png)

Antes de cambiar de red se mira la configuración del Zorin con `ip a`. La interfaz es **enp0s3**, tiene la IP **10.0.2.15** que da el NAT de VirtualBox y su MAC es **08:00:27:a8:06:a5**:

![ip a del Zorin en NAT](media/cap11.png)

Se abre Wireshark con `sudo wireshark`:

![Pantalla inicial de Wireshark](media/cap12.png)

---

## 8. Captura de la negociación DHCP

Se sigue este orden:

1. Se inicia la captura en la interfaz **enp0s3** y se aplica el filtro de visualización `dhcp`.
2. Sin apagar la máquina, se cambia el adaptador del Zorin de **NAT** a **Red interna** con el nombre `Internet`, el mismo que el servidor:

   ![Cambio del Zorin a Red interna](media/cap13.png)

3. Se fuerza la renovación de la IP con la herramienta gráfica: en la configuración de red del Zorin, se desactiva y se vuelve a activar la conexión cableada.

Resultado de la captura:

![Captura DHCP en Wireshark](media/cap14.png)

| Nº | Paquete | IP origen | IP destino |
|---|---|---|---|
| 37 | DHCP Request | 0.0.0.0 | 255.255.255.255 |
| 44 | DHCP **Discover** | 0.0.0.0 | 255.255.255.255 |
| 45 | DHCP **Offer** | 192.169.7.1 | 192.169.7.10 |
| 46 | DHCP **Request** | 0.0.0.0 | 255.255.255.255 |
| 47 | DHCP **ACK** | 192.169.7.1 | 192.169.7.10 |

Se ve el proceso completo **DORA** (Discover, Offer, Request, ACK):

- **Discover:** el cliente busca un servidor DHCP en la red.
- **Offer:** el servidor le ofrece una IP, la 192.169.7.10, la primera del pool.
- **Request:** el cliente acepta y pide formalmente esa IP.
- **ACK:** el servidor confirma la concesión.

El primer paquete (**37**, un Request) aparece antes del Discover porque el cliente intenta primero renovar la IP que tenía en NAT (10.0.2.15). El servidor Kea no le contesta, porque esa IP no pertenece a su subred, y entonces el cliente empieza el proceso normal desde el Discover.

---

## 9. Análisis de los paquetes: broadcast y unicast

Para cada paquete se han mirado los apartados **Ethernet II** (direcciones MAC) e **Internet Protocol Version 4** (direcciones IP).

**Discover (paquete 44):**

![Detalle del Discover](media/cap15.png)

**Offer (paquete 45):**

![Detalle del Offer](media/cap16.png)

**Request (paquete 46):**

![Detalle del Request](media/cap17.png)

**ACK (paquete 47):**

![Detalle del ACK](media/cap18.png)

### Resumen

| Paquete | MAC origen | MAC destino | IP origen | IP destino | Tipo MAC | Tipo IP |
|---|---|---|---|---|---|---|
| Discover | 08:00:27:a8:06:a5 (cliente) | ff:ff:ff:ff:ff:ff | 0.0.0.0 | 255.255.255.255 | **Broadcast** | **Broadcast** |
| Offer | 08:00:27:4d:84:e4 (servidor) | 08:00:27:a8:06:a5 (cliente) | 192.169.7.1 | 192.169.7.10 | **Unicast** | **Unicast** |
| Request | 08:00:27:a8:06:a5 (cliente) | ff:ff:ff:ff:ff:ff | 0.0.0.0 | 255.255.255.255 | **Broadcast** | **Broadcast** |
| ACK | 08:00:27:4d:84:e4 (servidor) | 08:00:27:a8:06:a5 (cliente) | 192.169.7.1 | 192.169.7.10 | **Unicast** | **Unicast** |

**Explicación:**

- Los paquetes que envía el **cliente** (Discover y Request) son **broadcast**, tanto en MAC como en IP. El cliente todavía no tiene IP (por eso usa 0.0.0.0 como origen) y no sabe dónde está el servidor, así que envía el mensaje a todos los equipos de la red. El Request también va en broadcast para que, si hubiera varios servidores DHCP, todos sepan qué oferta ha aceptado el cliente.
- Los paquetes que envía el **servidor** (Offer y ACK) son **unicast**, tanto en MAC como en IP. El servidor ya conoce la MAC del cliente porque venía en el Discover, así que le responde directamente a él, a la IP que le está ofreciendo.

---

## 10. Comprobación del cliente

Con `ip a` e `ip route` se comprueba que el Zorin se ha configurado por DHCP:

![ip a e ip route del Zorin](media/cap19.png)

- **IP:** 192.169.7.11/24, dentro del pool.
- **Puerta de enlace:** `default via 192.169.7.254 ... proto dhcp`, recibida por DHCP.
- **Tiempo de concesión:** `valid_lft 3941sec`, es decir, los 4000 segundos configurados, que van bajando.

> En la captura de Wireshark el cliente recibió la **.10**, pero en esta comprobación (hecha otro día, después de reiniciar las máquinas) recibió la **.11**. Kea todavía tenía la .10 registrada como concedida y le asignó la siguiente libre del pool. Es un comportamiento normal y la IP sigue estando dentro del rango.

En los detalles de la conexión cableada se ven también el **DNS 8.8.8.8** y la MAC del cliente:

![Detalles de la conexión cableada](media/cap20.png)

El cliente **no tiene acceso a Internet**, y es lo esperado: la puerta de enlace 192.169.7.254 no existe en la red interna. La práctica solo pide que el cliente reciba bien la configuración.

> **Problema encontrado:** al encender solo el Zorin, la conexión cableada no se conectaba. Era porque el Ubuntu Server estaba apagado y, en la red interna, es el único que reparte IP. Al encender el servidor, el cliente recibió la IP sin problemas.

---

## 11. Concesiones del servidor

Las concesiones se guardan en el archivo indicado en `lease-database`:

```bash
cat /var/lib/kea/kea-leases4.csv
```

> La presentación habla de `/var/lib/kea/dhcp4.leases`, pero en esta práctica el archivo es `kea-leases4.csv`, porque es el nombre que se puso en el `kea-dhcp4.conf`.

![Archivo de concesiones](media/cap21.png)

Cada línea es una concesión. Los campos más importantes son:

- **address:** la IP concedida, 192.169.7.11.
- **hwaddr:** la MAC del cliente, 08:00:27:a8:06:a5.
- **valid_lifetime:** la duración, 4000 segundos.
- **expire:** cuándo caduca (en formato de tiempo Unix).
- **subnet_id:** 1, el `id` de la subred del conf.
- **hostname:** el nombre del cliente, `fxsty-virtualbox`.

La misma IP aparece varias veces porque Kea añade una línea nueva cada vez que se renueva o modifica la concesión (por ejemplo, cada vez que se ha desactivado y activado la red del cliente). La línea con `valid_lifetime` 0 corresponde al momento en que el cliente liberó la IP al desconectarse.

---

## 12. Reserva de una IP para el cliente

Para que el Zorin reciba siempre la misma IP, se crea una **reserva** con su MAC (`08:00:27:a8:06:a5`) y la IP **192.169.7.55**. La reserva se pone dentro del bloque de la subred, pero **fuera del pool** (la .55 no está entre la .10 y la .50), para evitar conflictos con las IP que se reparten de forma dinámica.

```json
"pools": [ { "pool": "192.169.7.10 - 192.169.7.50" } ],
"reservations": [
  { "hw-address": "08:00:27:a8:06:a5", "ip-address": "192.169.7.55" }
],
```

Al añadir la reserva, hay que poner una **coma** después del `]` de `pools`, porque ya no es el último elemento del bloque.

Archivo completo con la reserva:

![kea-dhcp4.conf con la reserva](media/cap22.png)

Se comprueba la sintaxis, se reinicia el servicio y se verifica que está en marcha:

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server
```

![Sintaxis y estado después de la reserva](media/cap23.png)

En el Zorin se desactiva y se vuelve a activar la conexión de red para que pida otra vez la IP. Ahora recibe la **192.169.7.55**, la IP reservada:

![El Zorin con la IP reservada 192.169.7.55](media/cap24.png)

---

## 13. Conclusiones

- Se ha instalado y configurado un servidor DHCP con **Kea** en Ubuntu Server, que reparte IP, puerta de enlace y DNS a los clientes de la red interna 192.169.7.0/24.
- Con **Wireshark** se ha capturado el proceso **DORA** y se ha comprobado que los mensajes del cliente (Discover y Request) son **broadcast** y los del servidor (Offer y ACK) son **unicast**, tanto a nivel de MAC como de IP.
- Con una **reserva** se puede asignar siempre la misma IP a un equipo concreto a partir de su MAC.
- Al trabajar con el archivo JSON de Kea es muy importante revisar las llaves, los corchetes y las comas, y comprobar siempre la sintaxis con `kea-dhcp4 -t` antes de reiniciar el servicio.