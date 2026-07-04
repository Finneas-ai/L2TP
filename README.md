# VPN Client-to-Site con L2TP/IPsec (IKEv1)

Video de referencia: https://youtu.be/gxmWTlnS5bI

## Descripción

Acceso remoto **Client-to-Site** mediante **L2TP sobre IPsec (IKEv1)**. Un cliente Windows remoto se conecta a través de un router `Internet` (ISP) de tránsito hacia un `Servidor-VPN` Cisco IOS. IPsec en modo transporte protege el túnel L2TP, y PPP (dentro de L2TP) autentica al usuario y le asigna una IP de un pool interno.

## Topología

```
PC Linux (LAN) -- Servidor-VPN -- Internet (ISP) -- Cliente Windows (remoto)
```

## Direccionamiento IP

| Dispositivo        | Interfaz        | IP / Máscara                      |
|---------------------|-----------------|-------------------------------------|
| Internet (ISP)      | e0/0            | 200.6.82.2 /30                     |
| Internet (ISP)      | e0/1            | 200.6.83.1 /30                     |
| Internet (ISP)      | e0/2            | 200.6.84.1 /30                     |
| Servidor-VPN        | e0/0 (WAN)      | 200.6.82.1 /30                     |
| Servidor-VPN        | e0/1 (LAN)      | 172.6.82.1 /24                     |
| PC Linux            | eth0            | 172.6.82.10 /24, gw 172.6.82.1     |
| Cliente Windows     | -               | 200.6.83.2 /30, gw 200.6.83.1      |
| Cliente Windows     | Adaptador L2TP  | 172.6.82.100–172.6.82.110 (pool)   |

## Parámetros IKEv1 / IPsec / L2TP

| Parámetro       | Valor                                  |
|-----------------|------------------------------------------|
| encryption      | 3des                                     |
| hash            | sha                                       |
| group (DH)      | 2                                          |
| lifetime        | 86400 s                                   |
| PSK             | Cisco0682$ (address 0.0.0.0 0.0.0.0)     |
| transform-set   | esp-3des esp-sha-hmac, modo transport     |
| crypto map      | dynamic (DMAP-L2TP / CMAP-L2TP)          |
| pool L2TP       | 172.6.82.100–172.6.82.110                 |
| autenticación PPP | ms-chap-v2, chap                        |
| usuarios        | user1 / user2, password Usuario0682       |

## Configuración

### Internet (ISP)

```
hostname Internet

interface e0/0
 ip address 200.6.82.2 255.255.255.252
 no shutdown

interface e0/1
 ip address 200.6.83.1 255.255.255.252
 no shutdown

interface e0/2
 ip address 200.6.84.1 255.255.255.252
 no shutdown
```

### Servidor-VPN

```
hostname Servidor-VPN

!
! Interfaces
!
interface e0/0
 description WAN
 ip address 200.6.82.1 255.255.255.252
 no shutdown

interface e0/1
 description LAN
 ip address 172.6.82.1 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 200.6.82.2

!
! Usuarios VPN
!
username user1 password Usuario0682
username user2 password Usuario0682

!
! IKEv1
!
crypto isakmp policy 10
 encryption 3des
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key Cisco0682$ address 0.0.0.0 0.0.0.0

!
! IPSec
!
crypto ipsec transform-set TS-L2TP esp-3des esp-sha-hmac
 mode transport

crypto dynamic-map DMAP-L2TP 10
 set transform-set TS-L2TP
 set security-association lifetime seconds 3600

crypto map CMAP-L2TP 10 ipsec-isakmp dynamic DMAP-L2TP

interface e0/0
 crypto map CMAP-L2TP

!
! L2TP
!
vpdn enable

vpdn-group L2TP-GROUP
 accept-dialin
  protocol l2tp
  virtual-template 1
 no l2tp tunnel authentication

ip local pool L2TP-POOL 172.6.82.100 172.6.82.110

interface Virtual-Template1
 ip unnumbered e0/1
 peer default ip address pool L2TP-POOL
 ppp authentication ms-chap-v2 chap
```

### PC Linux

```
sudo ip addr add 172.6.82.10/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 172.6.82.1
```

### Cliente Windows

**Interfaz de red**

| Parámetro | Valor            |
|-----------|-------------------|
| IP        | 200.6.83.2        |
| Máscara   | 255.255.255.252   |
| Gateway   | 200.6.83.1        |

**Conexión VPN (integrada de Windows)**

| Parámetro     | Valor                                       |
|---------------|-----------------------------------------------|
| Proveedor     | Windows (integrado)                          |
| Nombre        | L2TP-LAB                                      |
| Servidor      | 200.6.82.1                                    |
| Tipo de VPN   | L2TP/IPsec con clave previamente compartida  |
| Clave PSK     | Cisco0682$                                    |
| Usuario       | user1                                         |
| Contraseña    | Usuario0682                                   |

> Si el cliente Windows está detrás de NAT, suele requerirse el ajuste de registro `AssumeUDPEncapsulationContextOnSendRule` (DWORD = 2) en `HKLM\SYSTEM\CurrentControlSet\Services\PolicyAgent`.

## Verificación

```
show crypto isakmp sa
show crypto ipsec sa
show vpdn session
show users / show ppp
ping 172.6.82.10   ; desde el cliente Windows, tras conectar la VPN
```
