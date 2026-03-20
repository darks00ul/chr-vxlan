# Extensión L2 sobre MPLS — Mikrotik VXLAN + IPsec

Extiende un segmento de Capa 2 entre dos sitios geográficamente separados sobre un enlace WAN MPLS o IP, utilizando **VXLAN** como encapsulamiento L2-sobre-L3 e **IPsec** para el cifrado. Ambos extremos son routers Mikrotik con RouterOS 7.x.

Este esquema es útil para escenarios de **Disaster Recovery** donde las VMs deben conservar sus direcciones IP originales al hacer failover hacia un sitio remoto, haciendo el failover transparente para los usuarios finales y los gateways upstream.

---

## Visión general de la arquitectura

```
Sitio A                        WAN (MPLS/IP)                   Sitio B
─────────────                  ─────────────                   ─────────────
VMs / hosts                                                    VMs / hosts
     │                                                               │
  Switch                                                          Switch
  (VLANs)                                                         (VLANs)
     │                                                               │
 Mikrotik A  ──── VXLAN (VNI) ──── IPsec ESP ────  Mikrotik B
 VTEP + IPsec      encapsulado       AES-256         VTEP + IPsec
```

Los dos routers Mikrotik actúan como **VTEPs** (Virtual Tunnel Endpoints) de VXLAN. Cada router hace bridge entre su(s) VLAN(s) local(es) y una interfaz VXLAN, extendiendo efectivamente el mismo dominio de broadcast L2 entre ambos sitios. IPsec en **modo transport** cifra todo el tráfico VXLAN entre las dos IPs WAN — el proveedor solo ve paquetes ESP y no puede inspeccionar el contenido.

### Decisiones de diseño

| Decisión | Motivo |
|---|---|
| VXLAN | Estándar RFC 7348 — interoperable con Linux, OVN y otras plataformas |
| IPsec en modo transport (no tunnel) | VXLAN ya maneja el tunneling — no se necesita doble encapsulación |
| Policy IPsec sin filtro de puerto | En RouterOS 7, VXLAN usa puertos de origen efímeros; filtrar por dst-port=4789 causó fallas en la negociación de Phase 2 durante las pruebas |
| Sin IP en las interfaces bridge | Los routers son transparentes a nivel L2 — invisibles para el segmento del cliente |
| Un VNI por VLAN | Aislamiento limpio; cada segmento mapea a un bridge independiente |

---

## Requisitos

- RouterOS **7.2 o superior** en ambos routers (subcomando `vxlan vteps`)
- Conectividad WAN entre las IPs WAN de ambos routers (MPLS, fibra dedicada o internet)
- UDP 4789 alcanzable entre las dos IPs WAN, o bien IPsec habilitado (UDP 500, UDP 4500, protocolo ESP 50)
- Los puertos de switch conectados a los routers configurados como **trunk 802.1q** si se usan múltiples VLANs

---

## Planificación de IPs (ejemplo)

```
Router sitio A WAN:   172.16.1.2/30   gateway: 172.16.1.1
Router sitio B WAN:   172.16.2.2/30   gateway: 172.16.2.1

Segmento del cliente: 10.0.0.0/24    (VLAN 10, VNI 10)
```

Adaptar las direcciones y valores de VNI al entorno propio. Las IPs WAN son las que actúan como direcciones VTEP.

---

## Configuración — VLAN única

### Router sitio A

```routeros
# ── Interfaz WAN ───────────────────────────────────────────────
/ip address add address=172.16.1.2/30 interface=ether1 comment="WAN"
/ip route add dst-address=0.0.0.0/0 gateway=172.16.1.1

# ── Interfaz VXLAN ─────────────────────────────────────────────
/interface vxlan add name=vxlan-vni10 vni=10 port=4789
/interface vxlan vteps add interface=vxlan-vni10 remote-ip=172.16.2.2

# ── Bridge L2 puro — sin IP ────────────────────────────────────
/interface bridge add name=br-vni10
/interface bridge port add bridge=br-vni10 interface=ether2
/interface bridge port add bridge=br-vni10 interface=vxlan-vni10

# ── IPsec ──────────────────────────────────────────────────────
/ip ipsec proposal add name=prop-vxlan \
    enc-algorithms=aes-256-cbc auth-algorithms=sha256 pfs-group=modp2048

/ip ipsec peer add name=peer-sitiob address=172.16.2.2 exchange-mode=ike2

/ip ipsec identity add peer=peer-sitiob auth-method=pre-shared-key \
    secret="ReemplazarConSecretoSeguro!"

/ip ipsec policy add \
    src-address=172.16.1.2/32 dst-address=172.16.2.2/32 \
    action=encrypt proposal=prop-vxlan peer=peer-sitiob tunnel=no

# ── Ajuste de MTU ──────────────────────────────────────────────
# Overhead VXLAN ~50 bytes + IPsec ESP ~60 bytes = ~110 bytes en total
/interface vxlan set vxlan-vni10 mtu=1390
/interface bridge set br-vni10 mtu=1390
```

### Router sitio B

```routeros
# ── Interfaz WAN ───────────────────────────────────────────────
/ip address add address=172.16.2.2/30 interface=ether1 comment="WAN"
/ip route add dst-address=0.0.0.0/0 gateway=172.16.2.1

# ── Interfaz VXLAN ─────────────────────────────────────────────
/interface vxlan add name=vxlan-vni10 vni=10 port=4789
/interface vxlan vteps add interface=vxlan-vni10 remote-ip=172.16.1.2

# ── Bridge L2 puro — sin IP ────────────────────────────────────
/interface bridge add name=br-vni10
/interface bridge port add bridge=br-vni10 interface=ether2
/interface bridge port add bridge=br-vni10 interface=vxlan-vni10

# ── IPsec ──────────────────────────────────────────────────────
/ip ipsec proposal add name=prop-vxlan \
    enc-algorithms=aes-256-cbc auth-algorithms=sha256 pfs-group=modp2048

/ip ipsec peer add name=peer-sitioa address=172.16.1.2 exchange-mode=ike2

/ip ipsec identity add peer=peer-sitioa auth-method=pre-shared-key \
    secret="ReemplazarConSecretoSeguro!"

/ip ipsec policy add \
    src-address=172.16.2.2/32 dst-address=172.16.1.2/32 \
    action=encrypt proposal=prop-vxlan peer=peer-sitioa tunnel=no

# ── Ajuste de MTU ──────────────────────────────────────────────
/interface vxlan set vxlan-vni10 mtu=1390
/interface bridge set br-vni10 mtu=1390
```

---

## Configuración multi-VLAN

Cada VLAN obtiene su propio VNI, interfaz VXLAN y bridge. La policy IPsec no necesita cambios — ya cubre todo el tráfico entre las dos IPs WAN.

| VLAN (sitio A) | VNI | VLAN (sitio B) | Segmento |
|---|---|---|---|
| 10 | 10 | 110 | 10.0.0.0/24 |
| 20 | 20 | 120 | 10.0.1.0/24 |
| 30 | 30 | 130 | 10.0.2.0/24 |

> Los IDs de VLAN no necesitan coincidir entre sitios. El VNI es lo que vincula los dos segmentos — cada lado puede usar el ID de VLAN local que necesite.

### Interfaces adicionales (ejecutar en ambos routers, adaptando los VLAN IDs)

```routeros
# ── Subinterfaces VLAN sobre el puerto trunk ───────────────────
/interface vlan add name=ether2-vlan10 interface=ether2 vlan-id=10
/interface vlan add name=ether2-vlan20 interface=ether2 vlan-id=20
/interface vlan add name=ether2-vlan30 interface=ether2 vlan-id=30

# ── Interfaces VXLAN ───────────────────────────────────────────
/interface vxlan add name=vxlan-vni10 vni=10 port=4789
/interface vxlan add name=vxlan-vni20 vni=20 port=4789
/interface vxlan add name=vxlan-vni30 vni=30 port=4789

/interface vxlan vteps add interface=vxlan-vni10 remote-ip=<IP-WAN-REMOTA>
/interface vxlan vteps add interface=vxlan-vni20 remote-ip=<IP-WAN-REMOTA>
/interface vxlan vteps add interface=vxlan-vni30 remote-ip=<IP-WAN-REMOTA>

# ── Bridges — uno por VNI ──────────────────────────────────────
/interface bridge add name=br-vni10
/interface bridge add name=br-vni20
/interface bridge add name=br-vni30

/interface bridge port add bridge=br-vni10 interface=ether2-vlan10
/interface bridge port add bridge=br-vni10 interface=vxlan-vni10

/interface bridge port add bridge=br-vni20 interface=ether2-vlan20
/interface bridge port add bridge=br-vni20 interface=vxlan-vni20

/interface bridge port add bridge=br-vni30 interface=ether2-vlan30
/interface bridge port add bridge=br-vni30 interface=vxlan-vni30
```

El puerto del switch que conecta al router debe estar configurado como **trunk 802.1q** pasando las VLANs correspondientes. 

---

## Verificación

### 1. Confirmar que IPsec Phase 1 y Phase 2 están activos

```routeros
/ip ipsec active-peers print
```

El output debe mostrar el peer con `state=established`. Si el peer no aparece, verificar que UDP 500 y UDP 4500 no estén bloqueados entre las dos IPs WAN.

```routeros
/ip ipsec installed-sa print
```

Debe mostrar al menos dos SAs (inbound + outbound) con estado `mature`. Si está vacío a pesar de que el peer está establecido, la policy IPsec no está matcheando el tráfico — verificar que `src-address` y `dst-address` coincidan exactamente con las IPs WAN.

### 2. Verificar contadores de tráfico VXLAN

```routeros
/interface print stats where name=vxlan-vni10
```

Luego de enviar tráfico a través del túnel, `rx-byte` y `tx-byte` deben incrementar. Si `tx-byte` incrementa pero `rx-byte` se mantiene en cero, el router remoto no está respondiendo — verificar su `remote-ip` en vteps y su policy IPsec.

### 3. Probar la extensión L2 — ping entre sitios

Desde un host en el sitio A, hacer ping a un host en el sitio B dentro del mismo segmento:

```
ping 10.0.0.X
```

Si responde, el dominio L2 está correctamente extendido. Ambos hosts comparten el mismo dominio de broadcast independientemente de su ubicación física.

### 4. Verificar que el ARP cruza el túnel

```routeros
/interface bridge host print
```

Luego de un ping exitoso, la tabla de hosts del bridge en cada router debe mostrar MACs aprendidas tanto en `ether2` (local) como en `vxlan-vniX` (remoto). Las MACs remotas aprendidas sobre la interfaz VXLAN confirman que las tramas L2 están atravesando el túnel.

### 5. Confirmar que IPsec está cifrando el tráfico

```routeros
/tool sniffer quick interface=ether1 count=30
```

Todos los paquetes capturados en la interfaz WAN deben mostrar protocolo **ESP** (50). Si se ven paquetes UDP 4789 en claro, la policy IPsec no se está aplicando — verificar las direcciones de origen y destino en la policy.

### 6. Validar el MTU

```routeros
/ping <IP-HOST-REMOTO> size=1400 do-not-fragment count=5
```

Si los pings funcionan con `size=1400` pero fallan con valores mayores, el MTU está correctamente limitando la fragmentación. Si fallan con 1400, reducir el MTU de las interfaces VXLAN y bridge a 1380 y reintentar.

---

## Troubleshooting

| Síntoma | Causa probable | Solución |
|---|---|---|
| `active-peers print` vacío | UDP 500/4500 bloqueado, IP del peer incorrecta o PSK no coincide | Verificar reglas de firewall, IP del peer y que ambos lados usen el mismo `secret` |
| Peer establecido pero `installed-sa` vacío | Las direcciones `src`/`dst` en la policy no coinciden con las IPs WAN reales | Corregir las direcciones en la policy en ambos lados |
| VXLAN `tx` incrementa pero `rx` se mantiene en 0 | El `remote-ip` del VTEP remoto es incorrecto, o el tráfico de retorno no matchea la policy IPsec | Verificar `remote-ip` y policy en el router remoto |
| Ping funciona con paquetes pequeños pero falla con grandes | MTU no ajustado tras agregar el overhead de VXLAN + IPsec | Configurar MTU en 1390 (o menor) en la interfaz VXLAN y en el bridge |
| Ping entre sitios falla pero IPsec y VXLAN parecen correctos | Puerto del switch hacia el router no está en modo trunk | Configurar el puerto del switch como trunk 802.1q |
| Error de sintaxis en RouterOS al agregar `local-address` en el comando `vxlan add` | RouterOS 7 movió la config de VTEPs al subcomando `/interface vxlan vteps` | Usar `vteps add` para la IP remota; la IP local se detecta automáticamente desde la tabla de ruteo |

---

## Notas de seguridad

- Reemplazar el PSK de ejemplo con un secreto generado aleatoriamente de al menos 32 caracteres antes de desplegar en producción.
- Considerar autenticación basada en certificados en lugar de PSK para entornos productivos con múltiples peers.
- La policy IPsec en esta guía cifra **todo el tráfico** entre las dos IPs WAN, no solo UDP 4789. Esto es intencional — filtrar por puerto de destino causó fallas en la negociación de Phase 2 en RouterOS 7 debido a los puertos de origen efímeros que usa el stack VXLAN.
- Los routers no tienen dirección IP en las interfaces bridge ni en las subinterfaces VLAN. Son transparentes a nivel L2 para el segmento del cliente — esto es por diseño.

---

## Licencia

MIT

