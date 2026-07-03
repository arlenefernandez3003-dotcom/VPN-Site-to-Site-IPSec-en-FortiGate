# 🔐 FortiGate — VPN Site-to-Site IPSec IKEv1

<div align="center">

**Instituto Tecnológico de las Américas — ITLA**  
Seguridad de Redes · Prof. Jonathan Esteban Rondón Corniel  
**Arlene Fernández Herrera · Matrícula: 2025-0730**

![FortiGate](https://img.shields.io/badge/Plataforma-FortiGate-red?style=for-the-badge&logo=fortinet)
![IPSec](https://img.shields.io/badge/Protocolo-IPSec%20IKEv1-blue?style=for-the-badge)
![Type](https://img.shields.io/badge/Tipo-Site--to--Site-brightgreen?style=for-the-badge)
![Platform](https://img.shields.io/badge/Entorno-PNetLab-blueviolet?style=for-the-badge)

</div>

---

## 📑 Tabla de Contenidos

1. [Objetivo](#1-objetivo)
2. [Topología](#2-topología)
3. [Direccionamiento IP](#3-direccionamiento-ip)
4. [Parámetros Configurados](#4-parámetros-configurados)
5. [Scripts de Configuración](#5-scripts-de-configuración)
6. [Verificación mediante GUI](#6-verificación-mediante-gui)
7. [Capturas de Pantalla](#7-capturas-de-pantalla)
8. [Video Demostrativo](#8-video-demostrativo)

---

## 1. Objetivo

Implementar y verificar una **VPN Site-to-Site IPSec IKEv1** entre dos firewalls **FortiGate** en PNetLab, que permita la comunicación cifrada entre dos redes LAN geográficamente separadas. La práctica cubre:

- Configuración de interfaces WAN (`port1`) y LAN (`port2`) en cada FortiGate.
- Establecimiento del túnel IPSec mediante **Phase 1** (IKEv1, DES-SHA1, DH Group 2) y **Phase 2** (selección de subredes LAN como tráfico interesante).
- Configuración de **rutas estáticas** hacia la red remota usando la interfaz de túnel VPN como dispositivo de salida.
- Creación de **políticas de firewall** bidireccionales (`LAN-a-VPN` y `VPN-a-LAN`) para permitir el tráfico entre sitios.
- Verificación del estado del túnel y prueba de conectividad mediante **traceroute entre hosts** a través de la GUI de FortiGate.

---

## 2. Topología

```
                              [ INTERNET / ISP ]
                               192.168.19.0/24
                              (Cloud/NAT: 192.168.19.2)
                                      │
                  ┌───────────────────┴───────────────────┐
                  │ port1: 192.168.19.5                    │ port1: 192.168.19.6
          ┌───────┴──────────┐                   ┌────────┴──────────┐
          │  FortiGate1      │◄═══ Túnel VPN ════►│  FortiGate2      │
          │  (Sitio A)       │    IPSec IKEv1     │  (Sitio B)       │
          └───────┬──────────┘                   └────────┬──────────┘
                  │ port2: 202.50.73.193/26               │ port2: 202.50.73.129/26
                  │                                       │
          ┌───────┴──────────┐                   ┌────────┴──────────┐
          │      SW1         │                   │       SW2         │
          └───────┬──────────┘                   └────────┬──────────┘
                  │                                       │
          ┌───────┴──────────┐                   ┌────────┴──────────┐
          │      PC1         │                   │       PC2         │
          │  202.50.73.194   │                   │   202.50.73.130   │
          └──────────────────┘                   └───────────────────┘
           ◄──── SITIO A ────►                   ◄──── SITIO B ─────►
           202.50.73.192/26                       202.50.73.128/26
```

> El túnel IPSec se negocia entre `192.168.19.5` (FG1 port1) y `192.168.19.6` (FG2 port1).  
> El tráfico entre `202.50.73.192/26` y `202.50.73.128/26` viaja cifrado por el segmento Cloud/ISP.

**Flujo del túnel:**
1. PC1 envía tráfico hacia PC2 → FortiGate1 consulta la Phase 2 (subredes interesantes).
2. Phase 1 negocia el canal IKEv1 entre `192.168.19.5` y `192.168.19.6`.
3. Phase 2 establece la SA IPSec → tráfico cifrado viaja por Cloud/NAT.
4. Política `VPN-a-LAN` en FG2 permite el tráfico hacia PC2.

---

## 3. Direccionamiento IP

### Interfaces de Dispositivos

| Dispositivo    | Interfaz | Dirección IP      | Máscara | Gateway        | Rol                         |
|----------------|----------|-------------------|---------|----------------|-----------------------------|
| Cloud (NAT)    | —        | 192.168.19.2      | /24     | —              | Gateway Cloud NAT (PNetLab) |
| **FG1 (SitioA)**| **port1**| **192.168.19.5**  | **/24** | 192.168.19.2   | WAN → Cloud (role: wan)     |
| **FG1 (SitioA)**| **port2**| **202.50.73.193** | **/26** | —              | LAN → SW1 (role: lan)       |
| **FG2 (SitioB)**| **port1**| **192.168.19.6**  | **/24** | 192.168.19.2   | WAN → Cloud (role: wan)     |
| **FG2 (SitioB)**| **port2**| **202.50.73.129** | **/26** | —              | LAN → SW2 (role: lan)       |
| SW1            | —        | —                 | —       | —              | Capa 2 Sitio A              |
| SW2            | —        | —                 | —       | —              | Capa 2 Sitio B              |
| PC1            | eth0     | 202.50.73.194     | /26     | 202.50.73.193  | Host Sitio A                |
| PC2            | eth0     | 202.50.73.130     | /26     | 202.50.73.129  | Host Sitio B                |

### Tabla de Subredes

| Subred             | Rango Utilizable              | Broadcast       | Uso              |
|--------------------|-------------------------------|-----------------|------------------|
| `192.168.19.0/24`  | 192.168.19.1 – 192.168.19.254 | 192.168.19.255  | Segmento WAN/ISP |
| `202.50.73.128/26` | 202.50.73.129 – 202.50.73.190 | 202.50.73.191   | LAN Sitio B      |
| `202.50.73.192/26` | 202.50.73.193 – 202.50.73.254 | 202.50.73.255   | LAN Sitio A      |

---

## 4. Parámetros Configurados

### Phase 1 — IKEv1

| Parámetro        | Valor            | Descripción                                    |
|------------------|------------------|------------------------------------------------|
| IKE Version      | 1                | Protocolo de negociación IKEv1                 |
| Proposal         | DES-SHA1         | Cifrado DES + integridad SHA1                  |
| DH Group         | 2 (1024-bit)     | Intercambio Diffie-Hellman                     |
| Peer Type        | any              | Acepta cualquier peer (sin restricción por IP) |
| Pre-Shared Key   | `ITLA2025Arlene` | Clave compartida idéntica en ambos FG          |
| Remote GW (FG1)  | 192.168.19.6     | IP WAN del peer remoto FG2                     |
| Remote GW (FG2)  | 192.168.19.5     | IP WAN del peer remoto FG1                     |

### Phase 2 — IPSec SA

| Parámetro         | FG1 (Sitio A)       | FG2 (Sitio B)       |
|-------------------|---------------------|---------------------|
| Phase1 Name       | `VPN-a-SitioB`      | `VPN-a-SitioA`      |
| Proposal          | DES-SHA1            | DES-SHA1            |
| Src Subnet (local)| 202.50.73.192/26    | 202.50.73.128/26    |
| Dst Subnet (remota)| 202.50.73.128/26   | 202.50.73.192/26    |

### Rutas Estáticas

| Router | Destino            | Device (interfaz VPN)  |
|--------|--------------------|------------------------|
| FG1    | 202.50.73.128/26   | `VPN-a-SitioB`         |
| FG2    | 202.50.73.192/26   | `VPN-a-SitioA`         |

### Políticas de Firewall

| FortiGate | Política      | Src Intf        | Dst Intf        | Acción  |
|-----------|---------------|-----------------|-----------------|---------|
| FG1       | `LAN-a-VPN`   | port2 (LAN)     | VPN-a-SitioB    | accept  |
| FG1       | `VPN-a-LAN`   | VPN-a-SitioB    | port2 (LAN)     | accept  |
| FG2       | `LAN-a-VPN`   | port2 (LAN)     | VPN-a-SitioA    | accept  |
| FG2       | `VPN-a-LAN`   | VPN-a-SitioA    | port2 (LAN)     | accept  |

---

## 5. Scripts de Configuración

### FortiGate1 — Sitio A

```fortios
! ══════════════════════════════════════════════════════════════
!  FortiGate1 — Sitio A | VPN Site-to-Site IPSec IKEv1
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

! ── Interfaces ──────────────────────────────────────────────
config system interface
    edit "port1"
        set mode static
        set ip 192.168.19.5 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port2"
        set mode static
        set ip 202.50.73.193 255.255.255.192
        set allowaccess ping http ssh
        set role lan
    next
end

! ── Ruta por defecto hacia Cloud/ISP ────────────────────────
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 192.168.19.2
        set device "port1"
    next
end

! ── VPN Phase 1 (IKEv1) ─────────────────────────────────────
config vpn ipsec phase1-interface
    edit "VPN-a-SitioB"
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set proposal des-sha1
        set dhgrp 2
        set remote-gw 192.168.19.6
        set psksecret ITLA2025Arlene
    next
end

! ── VPN Phase 2 (IPSec SA — subredes interesantes) ──────────
config vpn ipsec phase2-interface
    edit "VPN-a-SitioB-P2"
        set phase1name "VPN-a-SitioB"
        set proposal des-sha1
        set src-subnet 202.50.73.192 255.255.255.192
        set dst-subnet 202.50.73.128 255.255.255.192
    next
end

! ── Ruta estática hacia LAN remota vía túnel ────────────────
config router static
    edit 2
        set dst 202.50.73.128 255.255.255.192
        set device "VPN-a-SitioB"
    next
end

! ── Políticas de Firewall (bidireccional) ───────────────────
config firewall policy
    edit 1
        set name "LAN-a-VPN"
        set srcintf "port2"
        set dstintf "VPN-a-SitioB"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 2
        set name "VPN-a-LAN"
        set srcintf "VPN-a-SitioB"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end

! ── Logging en memoria ───────────────────────────────────────
config log memory setting
    set status enable
end
```

### FortiGate2 — Sitio B

```fortios
! ══════════════════════════════════════════════════════════════
!  FortiGate2 — Sitio B | VPN Site-to-Site IPSec IKEv1
!  Arlene Fernández Herrera · 2025-0730 | Seguridad de Redes
! ══════════════════════════════════════════════════════════════

! ── Interfaces ──────────────────────────────────────────────
config system interface
    edit "port1"
        set mode static
        set ip 192.168.19.6 255.255.255.0
        set allowaccess ping https ssh
        set role wan
    next
    edit "port2"
        set mode static
        set ip 202.50.73.129 255.255.255.192
        set allowaccess ping http ssh
        set role lan
    next
end

! ── Ruta por defecto hacia Cloud/ISP ────────────────────────
config router static
    edit 1
        set dst 0.0.0.0 0.0.0.0
        set gateway 192.168.19.2
        set device "port1"
    next
end

! ── VPN Phase 1 (IKEv1) ─────────────────────────────────────
config vpn ipsec phase1-interface
    edit "VPN-a-SitioA"
        set interface "port1"
        set ike-version 1
        set peertype any
        set net-device disable
        set proposal des-sha1
        set dhgrp 2
        set remote-gw 192.168.19.5
        set psksecret ITLA2025Arlene
    next
end

! ── VPN Phase 2 (IPSec SA — subredes interesantes) ──────────
config vpn ipsec phase2-interface
    edit "VPN-a-SitioA-P2"
        set phase1name "VPN-a-SitioA"
        set proposal des-sha1
        set src-subnet 202.50.73.128 255.255.255.192
        set dst-subnet 202.50.73.192 255.255.255.192
    next
end

! ── Ruta estática hacia LAN remota vía túnel ────────────────
config router static
    edit 2
        set dst 202.50.73.192 255.255.255.192
        set device "VPN-a-SitioA"
    next
end

! ── Políticas de Firewall (bidireccional) ───────────────────
config firewall policy
    edit 1
        set name "LAN-a-VPN"
        set srcintf "port2"
        set dstintf "VPN-a-SitioA"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 2
        set name "VPN-a-LAN"
        set srcintf "VPN-a-SitioA"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end

! ── Logging en memoria ───────────────────────────────────────
config log memory setting
    set status enable
end
```

### Hosts

```bash
# PC1 — Sitio A
ip 202.50.73.194 255.255.255.192 202.50.73.193

# PC2 — Sitio B
ip 202.50.73.130 255.255.255.192 202.50.73.129
```

---

## 6. Verificación mediante GUI

Toda la verificación de este laboratorio se realiza a través de la **interfaz gráfica (GUI)** del FortiGate accediendo por HTTPS a la IP de `port1` o `port2`.

### Estado del túnel VPN

**Ruta en GUI:** `VPN → IPsec Tunnels`

Aquí se visualiza el estado de cada túnel configurado. El indicador de color confirma el estado:

| Color | Estado |
|---|---|
| 🟢 Verde | Túnel activo — Phase 1 y Phase 2 establecidas |
| 🔴 Rojo | Túnel inactivo — negociación fallida o sin tráfico |
| ⚪ Gris | Túnel configurado pero sin intentar conectar |

Para activar el túnel manualmente desde la GUI, hacer clic en el botón **"Bring Up"** sobre el túnel correspondiente.

---

### Monitor de tráfico VPN

**Ruta en GUI:** `Monitor → IPsec Monitor`

Muestra en tiempo real:

- Nombre del túnel, IP del peer remoto y estado de Phase 1 / Phase 2.
- Bytes transmitidos y recibidos por el túnel.
- Tiempo activo de la SA (lifetime).

---

### Rutas estáticas activas

**Ruta en GUI:** `Network → Static Routes`

Confirmar que la ruta hacia la LAN remota existe y tiene la interfaz VPN como dispositivo de salida:

| Destino | Gateway | Interfaz |
|---|---|---|
| 202.50.73.128/26 | — | VPN-a-SitioB |
| 202.50.73.192/26 | — | VPN-a-SitioA |

---

### Políticas de firewall activas

**Ruta en GUI:** `Policy & Objects → Firewall Policy`

Verificar que existen las dos políticas en cada FortiGate y que su contador de bytes sea mayor que 0 tras generar tráfico, lo que confirma que el tráfico está fluyendo y siendo permitido.

---

### Traceroute entre LANs (prueba de conectividad)

**Ruta en GUI:** `Network → Diagnostics` → seleccionar **Traceroute**

Configurar:
- **Host:** `202.50.73.130` (IP de PC2 en Sitio B) desde FG1
- **Interface:** `port2` (LAN)

Resultado esperado:

```
traceroute to 202.50.73.130
 1  202.50.73.193    <1 ms    FortiGate1 LAN
 2  202.50.73.130    <5 ms    PC2 Sitio B (a través del túnel VPN)
```

> Si el traceroute llega a PC2 en 1 o 2 saltos pasando por la interfaz VPN, la comunicación entre LANs está funcionando correctamente.

---

### Log de eventos VPN

**Ruta en GUI:** `Log & Report → Events → VPN Events`

Muestra el historial de negociación IKE:

- `IPsec phase-1 status changed` → Phase 1 establecida.
- `IPsec phase-2 status changed` → Phase 2 establecida (SA activa).
- Errores de autenticación si la PSK no coincide.

---

## 7. Capturas de Pantalla

| # | Captura | Descripción |
|---|---|---|
| 1 | [Topología general](evidencias/1.png) | Topología en PNetLab con nombre y matrícula visibles, FG1, FG2 y router ISP encendidos. |
| 2 | [GUI VPN IPsec Tunnels](evidencias/2.png) | Panel `VPN → IPsec Tunnels` en FG1 mostrando el túnel `VPN-a-SitioB` en estado verde (activo). |
| 3 | [GUI IPsec Monitor](evidencias/3.png) | Panel `Monitor → IPsec Monitor` con Phase 1 y Phase 2 establecidas y bytes transmitidos. |
| 4 | [Traceroute exitoso](evidencias/4.png) | Resultado del traceroute desde FG1 hacia PC2 (`202.50.73.130`) confirmando comunicación entre LANs. |

---

## 8. Video Demostrativo

🎥 **[Ver en YouTube — enlace pendiente](https://youtu.be/bVAHTs_27zY)**



---

<div align="center">

**Arlene Fernández Herrera · 2025-0730 · ITLA**  
*Seguridad de Redes — Lab 08: FortiGate VPN Site-to-Site IPSec IKEv1*

</div>
