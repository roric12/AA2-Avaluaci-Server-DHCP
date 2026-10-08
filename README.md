# AA2 — Avaluació Server DHCP

Documentació de la pràctica del servei DHCP amb **Kea** sobre **Ubuntu 24.04 Server**, amb un client **Zorin OS** connectat per xarxa interna de VirtualBox.

---

## Objectius de l'activitat

L'activitat té com a finalitat muntar i validar un servei DHCP complet en un entorn virtualitzat, i entendre què passa realment a la xarxa quan un client obté la seva configuració.

| # | Objectiu | Descripció |
|---|---|---|
| 1 | Instal·lar el servei DHCP | Posar en marxa Kea sobre Ubuntu 24.04 Server, coneixent les unitats de systemd que el componen i els seus fitxers de configuració |
| 2 | Configurar la xarxa del servidor | Definir dues interfícies amb Netplan: una en NAT per a la sortida a Internet i una altra en xarxa interna dedicada al servei DHCP |
| 3 | Ajustar el servei als requisits | Definir una subxarxa, establir un rang d'adreces assignables i enviar als clients les opcions de porta d'enllaç i servidor de noms |
| 4 | Desactivar serveis no necessaris | Aturar DHCPv6 i DDNS, tant a nivell de systemd com dins del fitxer de configuració |
| 5 | Comprovar des d'un client real | Verificar que el client rep l'adreça, la màscara, la porta d'enllaç i el DNS esperats |
| 6 | Analitzar el protocol amb Wireshark | Capturar la negociació entre client i servidor i identificar cadascun dels quatre missatges que la componen |
| 7 | Distingir broadcast de unicast | Determinar quins paquets viatgen en difusió i quins en unicast, a nivell d'adreça IP i d'adreça MAC, i justificar-ne el motiu |
| 8 | Assignar adreces fixes per MAC | Definir reserves i entendre per què han de quedar fora del rang dinàmic |
| 9 | Interpretar els registres del servei | Consultar el fitxer de concessions i diagnosticar errors de configuració a partir de la sortida del sistema |

---

## Dades de la pràctica

| Dada | Valor |
|---|---|
| Número de llista | **2** |
| Subxarxa | `192.169.2.0/24` |
| IP del servidor (xarxa interna) | `192.169.2.1` |
| Pool DHCP | `192.169.2.10` – `192.169.2.50` |
| Porta d'enllaç anunciada | `192.169.2.254` |
| Servidor DNS anunciat | `8.8.8.8` |
| Reserva per MAC | `192.169.2.55` |
| MAC del client | `08:00:27:94:e7:b9` |

---

---

## Índex

1. [Escenari i topologia](#1-escenari-i-topologia)
2. [Configuració de xarxa del servidor](#2-configuració-de-xarxa-del-servidor)
3. [Instal·lació de Kea](#3-installació-de-kea)
4. [Desactivació de DHCPv6 i DDNS](#4-desactivació-de-dhcpv6-i-ddns)
5. [Configuració del servei DHCPv4](#5-configuració-del-servei-dhcpv4)
6. [Aplicació i verificació del servei](#6-aplicació-i-verificació-del-servei)
7. [Captura del trànsit amb Wireshark](#7-captura-del-trànsit-amb-wireshark)
8. [Anàlisi broadcast / unicast](#8-anàlisi-broadcast--unicast)
9. [Comprovació del client i reserva per MAC](#9-comprovació-del-client-i-reserva-per-mac)
10. [Conclusions](#10-conclusions)

---

## 1. Escenari i topologia

L'escenari consta de dues màquines virtuals allotjades al mateix amfitrió:

```
                 ┌─────────────────────────────┐
   Internet ─────┤ enp0s3  (NAT)               │
                 │                             │
                 │   SERVIDOR Ubuntu 24.04     │
                 │   Kea DHCPv4                │
                 │                             │
                 │ enp0s8  192.169.2.1/24      │
                 └──────────────┬──────────────┘
                                │  xarxa interna "intnet"
                 ┌──────────────┴──────────────┐
                 │ enp0s3  (DHCP)              │
                 │                             │
                 │   CLIENT Zorin OS           │
                 └─────────────────────────────┘
```

**Servidor Ubuntu Server**

| Adaptador | Mode | Funció |
|---|---|---|
| 1 — `enp0s3` | NAT | Accés a Internet (actualitzacions, paquets) |
| 2 — `enp0s8` | Xarxa interna `intnet` | Interfície on escolta el servei DHCP |

**Client Zorin OS**

| Adaptador | Mode | Funció |
|---|---|---|
| 1 — `enp0s3` | Xarxa interna `intnet` | Rep la configuració via DHCP |

> **Nota.** Durant la pràctica el servidor disposa d'una tercera interfície (`enp0s9`, xarxa només-amfitrió, `192.168.56.104`) que s'utilitza exclusivament per administrar la màquina per SSH des de l'amfitrió. No intervé en cap moment en el servei DHCP, que està lligat únicament a `enp0s8`.

El client arrenca inicialment en mode **NAT** per poder instal·lar Wireshark, i es commuta a **xarxa interna** en calent mentre la captura ja està en marxa (apartat 7).

---

## 2. Configuració de xarxa del servidor

La segona interfície es configura amb **Netplan**, amb una adreça estàtica i **sense porta d'enllaç ni servidor de noms**, tal com indica l'enunciat. Així la sortida a Internet continua fent-se per `enp0s3`.

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      addresses: [192.169.2.1/24]
  version: 2
```

Aplicació dels canvis:

```bash
sudo netplan apply
ip a
```

> **Treballant per SSH.** Si s'administra el servidor remotament, convé fer servir `sudo netplan try` en lloc d'`apply`: aplica la configuració i la reverteix automàticament passats 120 segons si no es confirma, de manera que un error en el YAML no deixa la màquina inaccessible.

![Fitxer de configuració de Netplan](imatges/01-netplan.png)

*Fitxer `/etc/netplan/00-installer-config.yaml` amb les dues interfícies definides.*

---

## 3. Instal·lació de Kea

Kea és el servidor DHCP que recomana Ubuntu des que `isc-dhcp-server` va quedar sense suport per part dels seus desenvolupadors.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install kea
```

Durant la instal·lació apareix el diàleg de configuració del **kea-ctrl-agent**, que demana establir una contrasenya per a l'accés via API. Se selecciona l'opció **`configured_random_password`**, ja que en treballar directament sobre el servidor no es farà servir la gestió remota.

Si cal repetir aquesta configuració:

```bash
sudo dpkg-reconfigure kea-ctrl-agent
```

### Serveis que proporciona Kea

| Unitat systemd | Fitxer de configuració | Servei |
|---|---|---|
| `kea-dhcp4-server` | `/etc/kea/kea-dhcp4.conf` | Servidor DHCP IPv4 |
| `kea-dhcp6-server` | `/etc/kea/kea-dhcp6.conf` | Servidor DHCP IPv6 |
| `kea-dhcp-ddns-server` | `/etc/kea/kea-dhcp-ddns.conf` | Servidor DDNS |
| `kea-ctrl-agent` | `/etc/kea/kea-ctrl-agent.conf` | Agent de control remot |

---

## 4. Desactivació de DHCPv6 i DDNS

L'enunciat demana desactivar els serveis de DHCPv6 i DDNS. Es fa a dos nivells.

### 4.1. A nivell de systemd

```bash
sudo systemctl stop kea-dhcp6-server
sudo systemctl disable kea-dhcp6-server

sudo systemctl stop kea-dhcp-ddns-server
sudo systemctl disable kea-dhcp-ddns-server
```

Verificació de l'estat de totes les unitats:

```bash
systemctl list-unit-files | grep kea
```

![Desactivació dels serveis DHCPv6 i DDNS](imatges/02-serveis-desactivats.png)

*`kea-dhcp6-server` i `kea-dhcp-ddns-server` queden com a `disabled`, mentre que `kea-dhcp4-server` es manté `enabled`.*

> El nom real de la unitat del servei DDNS en aquesta versió és **`kea-dhcp-ddns-server`**, no `kea-dhcp-ddns` com indica la documentació de les transparències.

### 4.2. Al fitxer de configuració

A més, dins de `kea-dhcp4.conf` s'hi afegeix el bloc que desactiva explícitament les actualitzacions DNS dinàmiques:

```json
"dhcp-ddns": {
    "enable-updates": false
},
```

---

## 5. Configuració del servei DHCPv4

El fitxer `/etc/kea/kea-dhcp4.conf` s'escriu sencer amb els paràmetres que demana l'enunciat.

```bash
sudo cp /etc/kea/kea-dhcp4.conf /etc/kea/kea-dhcp4.conf.bak
```

```json
{
"Dhcp4": {

    "interfaces-config": {
        "interfaces": [ "enp0s8" ]
    },

    "valid-lifetime": 4000,
    "renew-timer": 1000,
    "rebind-timer": 2000,

    "lease-database": {
        "type": "memfile",
        "persist": true,
        "name": "/var/lib/kea/kea-leases4.csv"
    },

    "dhcp-ddns": {
        "enable-updates": false
    },

    "subnet4": [
        {
            "id": 1,
            "subnet": "192.169.2.0/24",

            "option-data": [
                {
                    "name": "routers",
                    "data": "192.169.2.254"
                },
                {
                    "name": "domain-name-servers",
                    "data": "8.8.8.8"
                }
            ],

            "pools": [ { "pool": "192.169.2.10 - 192.169.2.50" } ],

            "reservations": [
                {
                    "hw-address": "08:00:27:94:e7:b9",
                    "ip-address": "192.169.2.55"
                }
            ]
        }
    ]
}
}
```

![Fitxer kea-dhcp4.conf](imatges/03-kea-dhcp4-conf.png)

*Contingut complet de `/etc/kea/kea-dhcp4.conf`.*

### Explicació dels paràmetres

| Clau | Valor | Significat |
|---|---|---|
| `interfaces-config` | `enp0s8` | Interfície per la qual escolta el servei. Si s'hi posés la de NAT, el servidor no sentiria mai les peticions del client. |
| `valid-lifetime` | `4000` | Durada en segons de la concessió. |
| `renew-timer` | `1000` | Moment en què el client intenta renovar la concessió (T1). |
| `rebind-timer` | `2000` | Moment en què el client busca qualsevol altre servidor si el seu no respon (T2). |
| `lease-database` | `memfile` | Les concessions es guarden en un fitxer CSV en lloc d'una base de dades. |
| `id` | `1` | Identificador de la subxarxa. **És obligatori** a la versió de Kea d'Ubuntu 24.04; sense ell el servei no arrenca. |
| `routers` | `192.169.2.254` | Porta d'enllaç que s'anuncia als clients. |
| `domain-name-servers` | `8.8.8.8` | Servidor DNS que s'anuncia als clients. |
| `pools` | `.10 – .50` | Rang d'adreces que es reparteixen dinàmicament. |
| `reservations` | `.55` | Adreça fixa lligada a una MAC concreta, **fora del pool** per evitar conflictes. |

> **Sintaxi JSON.** L'error més habitual en aquest fitxer són les comes: cada element en porta una excepte l'últim del seu bloc. En afegir `reservations` cal recordar posar una coma després del claudàtor de tancament de `pools`, ja que deixa de ser l'últim element de la subxarxa.

---

## 6. Aplicació i verificació del servei

Validació de la sintaxi abans de reiniciar res:

```bash
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf
```

Reinici i comprovació de l'estat:

```bash
sudo systemctl restart kea-dhcp4-server
sudo systemctl status kea-dhcp4-server --no-pager
```

![Validació i estat del servei](imatges/04-validacio-i-estat.png)

*La validació confirma que la subxarxa `192.169.2.0/24` s'ha afegit correctament amb els paràmetres `t1=1000`, `t2=2000` i `valid-lifetime=4000`. El servei apareix com a **active (running)** i escoltant a la interfície `enp0s8`.*

---

## 7. Captura del trànsit amb Wireshark

### 7.1. Preparació del client

El client Zorin s'arrenca inicialment amb l'adaptador en mode NAT per poder descarregar el paquet.

![Adaptador del client en mode NAT](imatges/05-client-nat.png)

*Configuració de xarxa del client a VirtualBox abans de començar la captura.*

Instal·lació i execució de Wireshark:

```bash
sudo apt install wireshark
sudo wireshark
```

### 7.2. Procediment de captura

1. S'inicia la captura sobre la interfície `enp0s3` del client.
2. **Amb la captura en marxa**, es canvia l'adaptador de **NAT** a **xarxa interna** des de *Dispositius → Xarxa* de VirtualBox.
3. Es força un refresc de la IP amb l'eina gràfica de l'escriptori (*Cablejat → Desconnectar* i tornar a connectar).
4. S'atura la captura i s'aplica el filtre de visualització `dhcp`.

![Llista de paquets DHCP capturats](imatges/06-wireshark-llista.png)

*Paquets DHCP capturats durant tot el procés.*

### 7.3. Paquets obtinguts

| # | Paquet | Origen | Destinació | Transacció |
|---|---|---|---|---|
| 3 | Request | `10.0.2.7` | `10.0.2.2` | `0x7595fd5` |
| 4 | ACK | `10.0.2.2` | `10.0.2.7` | `0x7595fd5` |
| 15 | Request | `0.0.0.0` | `255.255.255.255` | `0x506aca23` |
| 16 | NAK | `192.169.2.1` | `255.255.255.255` | `0x506aca23` |
| 17 | **Discover** | `0.0.0.0` | `255.255.255.255` | `0x4d6d5994` |
| 18 | **Offer** | `192.169.2.1` | `192.169.2.55` | `0x4d6d5994` |
| 19 | **Request** | `0.0.0.0` | `255.255.255.255` | `0x4d6d5994` |
| 20 | **ACK** | `192.169.2.1` | `192.169.2.55` | `0x4d6d5994` |

La captura mostra tres fases ben diferenciades:

- **Paquets 3 i 4** — Renovació de la concessió del NAT de VirtualBox, anterior al canvi de xarxa.
- **Paquets 15 i 16** — En passar a xarxa interna, el client intenta conservar la seva antiga adreça `10.0.2.7`. Com que aquesta no pertany a la subxarxa `192.169.2.0/24`, el servidor Kea la rebutja amb un **DHCPNAK**.
- **Paquets 17 a 20** — El rebuig obliga el client a reiniciar el procés des de zero, i es produeix la negociació completa **DORA** (Discover – Offer – Request – Acknowledge).

### 7.4. Detall dels paquets de la negociació

#### Paquet 17 — DHCP Discover

![Detall del paquet Discover](imatges/07-paquet-17-discover.png)

L'origen és `0.0.0.0` perquè el client encara no té cap adreça assignada, i la destinació és l'adreça de difusió limitada `255.255.255.255`. A nivell Ethernet, la MAC de destinació és `ff:ff:ff:ff:ff:ff`.

#### Paquet 18 — DHCP Offer

![Detall del paquet Offer](imatges/08-paquet-18-offer.png)

El servidor respon des de `192.169.2.1` i al camp **`Your (client) IP address`** ja hi consta `192.169.2.55`, cosa que confirma que la reserva per MAC s'està aplicant. El camp `Bootp flags: 0x0000 (Unicast)` indica que el client accepta rebre la resposta de forma unicast.

#### Paquet 19 — DHCP Request

![Detall del paquet Request](imatges/09-paquet-19-request.png)

El client confirma l'oferta rebuda. Continua enviant-se a `255.255.255.255`.

#### Paquet 20 — DHCP ACK

![Detall del paquet ACK](imatges/10-paquet-20-ack.png)

El servidor confirma definitivament l'assignació de `192.169.2.55` i la negociació queda tancada.

#### Paquet 3 — Request de renovació (NAT)

![Detall del paquet 3](imatges/11-paquet-3-renovacio.png)

Aquest paquet és anterior al canvi de xarxa i serveix de contrast: es tracta d'una **renovació** de concessió sobre el NAT de VirtualBox, no d'una negociació nova.

---

## 8. Anàlisi broadcast / unicast

| # | Paquet | MAC destinació | IP destinació | Nivell MAC | Nivell IP |
|---|---|---|---|---|---|
| 17 | Discover | `ff:ff:ff:ff:ff:ff` | `255.255.255.255` | **broadcast** | **broadcast** |
| 18 | Offer | `08:00:27:94:e7:b9` | `192.169.2.55` | unicast | unicast |
| 19 | Request | `ff:ff:ff:ff:ff:ff` | `255.255.255.255` | **broadcast** | **broadcast** |
| 20 | ACK | `08:00:27:94:e7:b9` | `192.169.2.55` | unicast | unicast |
| 3 | Request (renovació) | `08:00:27:2b:d0:2c` | `10.0.2.2` | unicast | unicast |

### Justificació

**Discover (17) — broadcast a les dues capes.**
El client acaba d'entrar a la xarxa: no té cap adreça IP assignada ni sap si hi ha algun servidor DHCP, ni molt menys quina és la seva adreça. L'única manera de fer arribar la petició és adreçar-la a tothom, de manera que s'envia amb la MAC de difusió `ff:ff:ff:ff:ff:ff` i amb la IP de difusió limitada `255.255.255.255`. Per això l'adreça d'origen és `0.0.0.0`.

**Offer (18) i ACK (20) — unicast a les dues capes.**
En aquest punt el servidor **ja coneix el client**, perquè la seva adreça MAC venia inclosa dins del Discover, i per tant pot respondre-li directament sense molestar la resta de la xarxa. Que les respostes vagin unicast també a nivell IP es deu al camp **`Bootp flags: 0x0000 (Unicast)`**: amb aquest valor el client indica que és capaç de processar una resposta unicast encara que no tingui l'adreça configurada. Si el flag fos `0x8000` (broadcast), el servidor hauria d'enviar-li les respostes per difusió.

**Request (19) — broadcast a les dues capes.**
Tot i que el client ja sap quina oferta vol acceptar i de quin servidor prové, el Request es torna a enviar per difusió. El motiu és que, si a la xarxa hi hagués més d'un servidor DHCP, tots han de poder sentir quina oferta ha estat acceptada: els que no han estat triats alliberen l'adreça que tenien reservada provisionalment i no la deixen bloquejada. El paquet inclou l'opció *Server Identifier* per indicar a quin servidor va dirigit.

**Request de renovació (3) — unicast a les dues capes.**
És un bon contrast amb el cas anterior. En una renovació, el client ja disposa d'una IP vàlida i ja coneix l'adreça del servidor que l'hi va concedir, de manera que pot dirigir-s'hi directament. No hi ha cap necessitat de difusió perquè no s'està triant entre diverses ofertes.

### Resum del comportament

```
CLIENT                                           SERVIDOR
  │                                                  │
  ├──── DISCOVER ─── broadcast MAC + IP ────────────>│
  │                                                  │
  │<─────────────── unicast MAC + IP ──── OFFER ─────┤
  │                                                  │
  ├──── REQUEST ──── broadcast MAC + IP ────────────>│
  │                                                  │
  │<─────────────── unicast MAC + IP ──── ACK ───────┤
  │                                                  │
```

La regla general és que **el client difon mentre no té adreça i el servidor respon directament perquè ja sap a qui s'adreça**, amb l'excepció del Request, que es difon per coordinar-se amb altres possibles servidors.

---

## 9. Comprovació del client i reserva per MAC

### 9.1. Reserva per adreça MAC

La reserva es defineix **dins del bloc de la subxarxa però fora del pool**, indicant l'adreça MAC del client:

```json
"reservations": [
    {
        "hw-address": "08:00:27:94:e7:b9",
        "ip-address": "192.169.2.55"
    }
]
```

L'adreça `192.169.2.55` queda fora del rang `.10 – .50`, de manera que no pot entrar en conflicte amb cap assignació dinàmica.

Després d'editar el fitxer cal reiniciar el servei i forçar la renovació de l'adreça al client:

```bash
sudo systemctl restart kea-dhcp4-server
```

```bash
# Al client
nmcli connection down "Wired connection 1" && nmcli connection up "Wired connection 1"
```

### 9.2. Resultat al client

![Configuració final del client i concessions del servidor](imatges/12-comprovacio-final.png)

Al client s'obté:

```
inet 192.169.2.55/24 ... dynamic enp0s3
    valid_lft 3190sec

default via 192.169.2.254 dev enp0s3 proto dhcp src 192.169.2.55 metric 20100
192.169.2.0/24 dev enp0s3 proto kernel scope link src 192.169.2.55 metric 100
```

Es comprova que:

- L'adreça assignada és exactament la reservada: **`192.169.2.55`**
- La porta d'enllaç anunciada és **`192.169.2.254`**, tal com demana l'enunciat
- La concessió té el temps de vida configurat al servidor

### 9.3. Fitxer de concessions del servidor

```bash
sudo cat /var/lib/kea/kea-leases4.csv
```

```
address,hwaddr,client_id,valid_lifetime,expire,subnet_id,fqdn_fwd,fqdn_rev,hostname,state,user_context,pool_id
192.169.2.55,08:00:27:94:e7:b9,01:08:00:27:94:e7:b9,4000,1791481130,1,0,0,usuari-virtualbox,0,,0
```

El registre confirma l'adreça assignada, la MAC del client, el temps de concessió de 4000 segons i el nom de la màquina client.

> L'enunciat fa referència a `/var/lib/kea/dhcp4.leases`, que és el nom per defecte del fitxer. En aquesta configuració s'ha definit explícitament com a `kea-leases4.csv` dins del bloc `lease-database`, seguint l'exemple de les transparències.

### 9.4. Estat final del servidor

![Interfícies del servidor](imatges/13-servidor-ip-a.png)

El servidor manté la interfície `enp0s3` amb l'adreça del NAT i `enp0s8` amb l'adreça estàtica `192.169.2.1/24` des de la qual dona servei DHCP.

---

## 10. Conclusions

La pràctica ha permès comprovar el funcionament complet del protocol DHCP en un entorn controlat:

- **La negociació DORA** es verifica paquet a paquet amb Wireshark, i s'observa com el client passa de no tenir cap adreça a rebre una configuració completa en quatre missatges.
- **L'ús de broadcast i unicast** no és arbitrari: respon a si l'emissor coneix o no el destinatari, i a la necessitat de coordinar diversos servidors DHCP en una mateixa xarxa.
- **El DHCPNAK** observat a la captura il·lustra què passa quan un client demana una adreça que no pertany a la subxarxa del servidor, i com aquest rebuig força el reinici de tot el procés.
- **Les reserves per MAC** permeten garantir que un equip concret rebi sempre la mateixa adreça sense renunciar als avantatges de la configuració automàtica.

### Problemes trobats i solucions

| Problema | Causa | Solució |
|---|---|---|
| `Device 'ens33' not found` | El nom de la interfície de les transparències no coincideix amb el del sistema | Consultar el nom real amb `nmcli device` o `ip a` |
| El servei no arrencava | Faltava la clau `"id"` a la subxarxa | Afegir `"id": 1` dins del bloc `subnet4` |
| `syntax error, unexpected end of file` | Claus o claudàtors sense tancar al JSON | Validar amb `kea-dhcp4 -t` abans de reiniciar |
| `Unit kea-dhcp-ddns.service not loaded` | El nom real de la unitat és diferent | Localitzar-la amb `systemctl list-unit-files \| grep kea` |

---

## Comandes de referència

```bash
# Xarxa
ip a                                              # Veure interfícies i adreces
ip r                                              # Veure taula d'encaminament
sudo netplan try                                  # Aplicar xarxa de forma reversible
sudo netplan apply                                # Aplicar xarxa

# Servei Kea
sudo kea-dhcp4 -t /etc/kea/kea-dhcp4.conf         # Validar la sintaxi
sudo systemctl restart kea-dhcp4-server           # Reiniciar el servei
sudo systemctl status kea-dhcp4-server            # Consultar l'estat
sudo journalctl -u kea-dhcp4-server -f            # Seguir el registre en directe
systemctl list-unit-files | grep kea              # Llistar les unitats de Kea

# Concessions
sudo cat /var/lib/kea/kea-leases4.csv             # Veure les concessions actives

# Client
nmcli device                                      # Llistar interfícies
nmcli connection down "Wired connection 1"        # Desactivar la connexió
nmcli connection up "Wired connection 1"          # Reactivar i renovar la IP
```

---

## Documentació consultada

- [Ubuntu Server Docs — How to install and configure isc-kea](https://ubuntu.com/server/docs/how-to-install-and-configure-isc-kea)
- [KEA — The DHCPv4 Server](https://kea.readthedocs.io/en/kea-1.6.2/arm/dhcp4-srv.html)
