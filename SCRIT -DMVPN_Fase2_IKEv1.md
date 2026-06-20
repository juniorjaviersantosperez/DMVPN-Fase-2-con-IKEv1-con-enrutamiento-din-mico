# DMVPN Fase 2 — IKEv1

**Topología:** R1 = Hub · R2 = Spoke1 · R3 = Spoke2  
**Enrutamiento dinámico:** EIGRP AS 100  
**Túnel mGRE:** `172.16.0.0/24` — Hub=`.1` · R2=`.2` · R3=`.3`

---

## Direccionamiento

| Dispositivo | Interfaz | IP             | Rol            |
|-------------|----------|----------------|----------------|
| R1 Hub      | e0/0     | 200.1.15.1/24  | NBMA fuente Tunnel10 |
| R1 Hub      | e0/1     | 200.1.99.1/24  | Tránsito a R3  |
| R2 Spoke1   | e0/0     | 200.1.15.2/24  | NBMA           |
| R2 Spoke1   | e0/1     | 10.15.99.1/24  | LAN            |
| R3 Spoke2   | e0/0     | 200.1.99.2/24  | NBMA           |
| R3 Spoke2   | e0/1     | 192.168.99.1/24| LAN            |
| Linux6      | e0       | 10.15.99.2/24  | Host LAN R2    |
| VPC8        | eth0     | 10.15.99.3/24  | Host LAN R2    |
| Linux7      | e0       | 192.168.99.2/24| Host LAN R3    |
| VPC9        | eth0     | 192.168.99.3/24| Host LAN R3    |

---

## Paso 1 — Interfaces físicas y rutas

### R1 Hub
```
conf t
interface e0/0
 ip address 200.1.15.1 255.255.255.0
 no shut
interface e0/1
 ip address 200.1.99.1 255.255.255.0
 no shut
```

### R2 Spoke1
```
conf t
interface e0/0
 ip address 200.1.15.2 255.255.255.0
 no shut
interface e0/1
 ip address 10.15.99.1 255.255.255.0
 no shut
ip route 0.0.0.0 0.0.0.0 200.1.15.1
```

### R3 Spoke2
```
conf t
interface e0/0
 ip address 200.1.99.2 255.255.255.0
 no shut
interface e0/1
 ip address 192.168.99.1 255.255.255.0
 no shut
ip route 0.0.0.0 0.0.0.0 200.1.99.1
```

---

## Paso 2 — IKEv1 Fase 1 (ISAKMP)

> El Hub usa `address 0.0.0.0 0.0.0.0` para aceptar cualquier Spoke.  
> Los Spokes apuntan a la IP NBMA del Hub (`200.1.15.1`).

### R1 Hub
```
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key cisco123 address 0.0.0.0 0.0.0.0
```

### R2 Spoke1
```
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key cisco123 address 200.1.15.1
```

### R3 Spoke2
```
crypto isakmp policy 10
 encr aes
 hash sha
 authentication pre-share
 group 2
 lifetime 86400

crypto isakmp key cisco123 address 200.1.15.1
```

---

## Paso 3 — IPSec Fase 2 (transform-set)

> Modo `transport`: IPSec protege el paquete GRE sin re-encapsular.

### R1, R2 y R3 (igual en todos)
```
crypto ipsec transform-set DMVPN-SET esp-aes esp-sha-hmac
 mode transport
```

---

## Paso 4 — Perfil IPSec

### R1, R2 y R3 (igual en todos)
```
crypto ipsec profile DMVPN-PROF
 set transform-set DMVPN-SET
```

---

## Paso 5 — Interfaz Tunnel10 mGRE + NHRP

> **Fase 2:** el Hub NO usa `ip nhrp redirect`.  
> Los Spokes NO usan `ip nhrp shortcut`.  
> El Hub no debe resumir rutas sobre Tunnel10 para que los Spokes vean el next-hop real y puedan crear túneles directos Spoke↔Spoke.

### R1 Hub
```
interface Tunnel10
 ip address 172.16.0.1 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 ip nhrp map multicast dynamic
 ip nhrp authentication dmvpn1
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

### R2 Spoke1
```
interface Tunnel10
 ip address 172.16.0.2 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 ip nhrp authentication dmvpn1
 ip nhrp nhs 172.16.0.1
 ip nhrp map 172.16.0.1 200.1.15.1
 ip nhrp map multicast 200.1.15.1
 ip nhrp registration no-unique
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

### R3 Spoke2
```
interface Tunnel10
 ip address 172.16.0.3 255.255.255.0
 no ip redirects
 ip nhrp network-id 100
 ip nhrp authentication dmvpn1
 ip nhrp nhs 172.16.0.1
 ip nhrp map 172.16.0.1 200.1.15.1
 ip nhrp map multicast 200.1.15.1
 ip nhrp registration no-unique
 tunnel source e0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROF
```

---

## Paso 6 — EIGRP dinámico

> **Obligatorio en R1:** `no ip split-horizon eigrp 100` sobre Tunnel10.  
> **Obligatorio en R1:** no resumir rutas sobre Tunnel10 (los Spokes necesitan el next-hop real del destino para el túnel directo).

### R1 Hub
```
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 10.15.99.0 0.0.0.255
 network 192.168.99.0 0.0.0.255
 no auto-summary

interface Tunnel10
 no ip split-horizon eigrp 100
 no ip summary-address eigrp 100 172.16.0.0 255.255.255.0
```

### R2 Spoke1
```
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 10.15.99.0 0.0.0.255
 no auto-summary
```

### R3 Spoke2
```
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 192.168.99.0 0.0.0.255
 no auto-summary
```

---

## Paso 7 — Hosts

### Linux6 (e0)
```
ip addr add 10.15.99.2/24 dev eth0
ip route add default via 10.15.99.1
```

### Linux7 (e0)
```
ip addr add 192.168.99.2/24 dev eth0
ip route add default via 192.168.99.1
```

### VPC8 (eth0)
```
ip 10.15.99.3 255.255.255.0 10.15.99.1
```

### VPC9 (eth0)
```
ip 192.168.99.3 255.255.255.0 192.168.99.1
```

---

## Paso 8 — Verificación

### Estado NHRP y túnel
```
show dmvpn
show ip nhrp
show interface Tunnel10
```

**Resultado esperado en R1:**
```
172.16.0.2/32 via 172.16.0.2 — Type: dynamic, Flags: registered nhop — NBMA: 200.1.15.2
172.16.0.3/32 via 172.16.0.3 — Type: dynamic, Flags: registered nhop — NBMA: 200.1.99.2
```

### Estado IKEv1
```
show crypto isakmp sa
```
Debe mostrar `QM_IDLE` para cada Spoke.

### IPSec
```
show crypto ipsec sa
```
Los contadores `encaps/decaps` deben incrementar con tráfico.

### EIGRP
```
show ip eigrp neighbors
show ip route eigrp
```
R1 debe ver a R2 y R3 como vecinos y aprender ambas LANs.

### Conectividad
```
ping 192.168.99.1 source 10.15.99.1
ping 192.168.99.2 source 10.15.99.1
ping 192.168.99.3 source 10.15.99.1
ping 10.15.99.2 source 192.168.99.1
```

---

## Notas importantes — Fase 2

- Los túneles Spoke↔Spoke se crean **bajo demanda** al primer paquete de tráfico.
- El Hub **no debe resumir** rutas sobre Tunnel10: los Spokes necesitan ver el next-hop real del Spoke destino para construir el túnel directo.
- Si `show ip nhrp` muestra `Type: incomplete, Flags: negative`, el Spoke no se registró. Ejecutar `clear ip nhrp <IP>` en el Hub y verificar autenticación NHRP y `tunnel source`.
- Todos los routers deben tener exactamente la misma `ip nhrp authentication` y el mismo `ip nhrp network-id`.
