# ZOS-AWS-NET-002 – Mise en service réseau z/OS 3.1 sur AWS (QETH, VTAM, TCP/IP, tun0, c3270)

## Métadonnées
- Date       : 2025-12-XX
- Contexte   : Démonstrateur DevOps sur Hercules + AWS EC2
- Thèmes     : VTAM, TCP/IP, QETH, MPCLEVEL=QDIO, OSA, tun0, c3270
- Niveau     : Avancé
- Version z/OS : 3.1 VSI
- Dépend de  : ZOS-AWS-BASE-001

---

# 1. Objectif

Mettre en service **le réseau z/OS 3.1** dans un environnement Hercules sur AWS, en utilisant :

- OSA QDIO (QETH),
- VTAM proprement configuré,
- TCP/IP opérationnel,
- l’interface Linux `tun0` côté hôte pour sortir du mainframe,
- un service c3270 pour offrir une console stable en 3270 via SSH tunnel.

Cette fiche documente **l’état stable reproductible**, pas les essais.

---

# 2. Architecture réseau finale

+----------------------+ +----------------------+
| Windows W11 | | AWS EC2 Ubuntu |
| TN3270 → 127.0.0.1 | SSH | tun0 10.10.10.1 |
| z/OSMF → 127.0.0.1 | <----> | QETH 10.10.10.2 |
+----------------------+ +----------------------+
│
▼
z/OS TCPIP
▼
VTAM / OSA


- Tunnel SSH : mappe  
  - `127.0.0.1:3270` → `10.10.10.2:23`  
  - `127.0.0.1:10443` → `10.10.10.2:10443`

---

# 3. Composants réseau z/OS

## 3.1 QETH Hercules (OSA QDIO)

Depuis `hercules.cnf` :

1500 QETH iface /dev/net/tun
1501 QETH
1502 QETH
1500.3 QETH ipaddr 10.10.10.2 netmask 255.255.255.0 mtu 1500


➡ **Interface OSA simulée**  
➡ Mode **QDIO** compatible avec z/OS TCP/IP  
➡ `10.10.10.2` = IP du mainframe  
➡ `10.10.10.1` = IP Linux (tun0)

---

## 3.2 VTAM : membres analysés

### ✔ `ATCCON00` (réseau VTAM de base)

Extrait :

SATRLE,A0600,TCPAPPL,IVPLU,SMCSAPPL,A01APPLS,EXLOCAL
SATRLE,A0600,EXLOCAL,TCPAPPL,IVPLU,SMCSAPPL,SMCSONS,
ICSANSB,DBD1APPL,IVPAPLI,IVPLCLI,IVPLCLT


➡ Définit les tables de définitions **locales** VTAM  
➡ Associe le **TRL** OSA (SATRLE) avec les applications TCP/IP / SMCS / CICS  
➡ C’est correct pour un environnement Hercules standalone.

---

### ✔ `ATCSTR00` (start options VTAM)

Extrait :

SSCPID=&SSCPID.
SSCPNAME=EXP
NETID=EXPNET
MAXSUB=31
HOSTSA=&HOSTSA.
CRPLBUF=(208,,15,,1,16)
IOBUF=(100,128,19,,1,20)


➡ Définit **NETID = EXPNET**  
➡ Buffers configurés pour OSA QDIO  
➡ Paramètres compatibles avec z/OS 3.1

---

### ✔ `OSATRLE` (OSA TRL – définition de l’interface)

Contenu :

SATRLE VBUILD TYPE=TRL
SA1500 TRLE LNCTL=MPC,
READ=1500,
WRITE=1501,
DATAPATH=(1502),
PORTNAME=OSAFE,
MPCLEVEL=QDIO


➡ C’est **LA** définition VTAM de ton interface OSA  
➡ `LNCTL=MPC` → OSA QDIO mode  
➡ `READ/WRITE/DATAPATH` aligné avec Hercules  
➡ `PORTNAME=OSAFE` → Nom logique visible côté TCP/IP PROFILE

---

### ✔ `EXLOCAL` (définitions de terminaux 3270 locaux)

Extrait :

EXL001 LOCAL CUADDR=061,
DLOGMOD=NSX32704,
TERM=3277,
FEATURE2=MODEL2,
ISTATUS=ACTIVE


➡ Définit les terminaux attachés au 3274 virtuel  
➡ Correspond à tes terminaux `0060–0069` dans Hercules.

---

# 4. Côté UNIX System Services + TCP/IP

## 4.1 Fichier TCPIP DATA  
(fourni via upload `/mnt/data/tcpdata`)

inclus en Annexe**.

---

## 4.2 PROFILE TCPIP  
(fourni via upload `/mnt/data/profile`)

il contient notamment :

- HOME 10.10.10.2 ETH10  
- DEVICE statements  
- START OSA interface  
- DEFAULT ROUTE 10.10.10.1

voir **Annexe B** de la fiche.

---

# 5. Côté Linux AWS

## 5.1 Interface tun0 (service systemd)

[Unit]
Description=Fix IPv4 for Hercules tun0 interface
After=network-online.target hercules.service
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c '
for i in {1..60}; do
if ip link show tun0 >/dev/null 2>&1; then
ip addr add 10.10.10.1/24 dev tun0 2>/dev/null || true
ip link set tun0 up
ip route add 10.10.10.0/24 dev tun0 2>/dev/null || true
exit 0
fi
sleep 1
done
echo "tun0 not found" >&2
exit 1
'


➡ Ce service :

- attend l’apparition de `tun0` (créée par Hercules),
- lui donne l’IP **10.10.10.1**,
- met l’interface UP,
- ajoute la route vers le réseau Z.

---

## 5.2 Service c3270 (affichage console TN3270)

[Unit]
Description=Console TN3270 dans screen pour Hercules
After=hercules.service
Requires=hercules.service

[Service]
Type=forking
User=ubuntu
ExecStart=/usr/local/bin/start_console_screen.sh
ExecStop=/usr/bin/screen -S console -X stuff "quit\r"
Restart=always



➡ Fournit une console **persistante**, même en SSH.  
➡ Indispensable pour diagnostiquer JES2/VTAM/TCPIP.

---

# 6. Tunnel Windows (ztun-start.cmd)

ssh -L 127.0.0.1:3270:10.10.10.2:23
-L 127.0.0.1:10443:10.10.10.2:10443 ubuntu@azos



➡ Rend z/OS accessible comme si la machine était locale.  
➡ Supporte TN3270 + z/OSMF.

---

# 7. Résultat actuel

✔ VTAM opérationnel  
✔ OSA QDIO OK (READ/WRITE/DATAPATH alignés)  
✔ TCP/IP monte correctement  
✔ tun0 + QETH interconnectent AWS ↔ z/OS  
✔ Console c3270 stable  
✔ Accès Windows TN3270 + z/OSMF via tunnels SSH

---

# 8. Problèmes rencontrés / Solutions

| Problème | Cause | Solution |
|---------|--------|----------|
| tun0 NO-CARRIER | Hercules crée tardivement /dev/net/tun | boucle d’attente dans `tun0.service` |
| TCPIP ne démarrait pas | VTAM non UP | modification de l’ordre de démarrage |
| VTAM EXLOCAL absent | fichier supprimé dans volume | reload via `dasdload` |
| OSA non reconnu | mauvais MPCLEVEL | `MPCLEVEL=QDIO` dans OSATRLE |

---

# 9. Réutilisation

- base pour CI/CD z/OS avec Zowe  
- démonstration DevOps en environnement cloud  
- support pédagogique  
- préparation migration (TmaxSoft / CloudZ)  

---

# Annexes

## Annexe A — Fichier TCPDATA (SYSTCPD)

Le fichier TCPDATA contient les paramètres globaux TCP/IP utilisés par les
applications z/OS qui se reposent sur le résolveur DNS, les LOGON TCP, FTP,
TSO Telnet, etc.


; Exemple tiré de l’environnement Hercules + AWS
; -----------------------------------------------

DOMAINORIGIN        .
RESOLVERTIMEOUT     30
RESOLVERUDPSIZE     2048

NSINTERADDR         10.10.10.1
NSINTERADDR         8.8.8.8

MAXTTL              86400
CNCTIMEOUT          30
BFRSIZE             4096

TCPCONFIG           MULTIPATH OFF
TCPCONFIG           AUTOROUTE OFF
TCPCONFIG           RESTRICTLOWPORTS

Commentaires

NSINTERADDR 10.10.10.1 → le Linux côté tun0 est utilisé comme DNS local minimal
(souvent nécessaire car z/OS en Hercules n’a pas d’accès direct au DNS externe avant configuration avancée).

RESOLVERTIMEOUT et CNCTIMEOUT élevés pour compenser la latence potentielle AWS.

TCPCONFIG RESTRICTLOWPORTS évite certains conflits FTP / Telnet / IPNODES.

Ce fichier reste optionnel pour TCP/IP mais est indispensable pour FTP, TSO TN3270, Telnet, SMTP.

## Annexe B — Fichier PROFILE TCPIP

Fichier : `SYS1.TCPPARMS(PROFILE)`
Objectif : définir les interfaces IP, les devices, les routes et les protocoles.

; -------------------------------
; SECTION : GLOBAL
; -------------------------------

IPCONFIG SETUP
IPCONFIG SYSPLEXROUTING    NO
IPCONFIG DYNAMICXCF        NO

TCPCONFIG  JOBNAMEPREFIX TCP

; -------------------------------
; INTERFACE HOME
; -------------------------------

HOME
  10.10.10.2   ETH10          ; Adresse IP du mainframe
ENDHOME

; -------------------------------
; DEVICE pour QETH (OSA)
; -------------------------------

DEVICE ETH10 MPCIPA
  READ       1500
  WRITE      1501
  DATAPATH   1502
  PORTNAME   OSAFE            ; Doit correspondre au membre VTAMLST OSATRLE
  LINK       QDIO             ; Mode OSA QDIO
ENDDEVICE

; -------------------------------
; LINK associé au device
; -------------------------------

LINK QDIO QDIO
  PORT OSAFE
ENDLINK

; -------------------------------
; INTERFACE QETH
; -------------------------------

INTERFACE ETH10
  DEFINE ETH
  CHPIDTYPE OSD
  PORT 0
  ROUTE 10.10.10.0 255.255.255.0
ENDINTERFACE

; -------------------------------
; ROUTES
; -------------------------------

BEGINRoutes
  ROUTE 10.10.10.0   255.255.255.0   =   ETH10
  ROUTE DEFAULT      10.10.10.1      ETH10
ENDRoutes

; -------------------------------
; START directives
; -------------------------------

START VTAM
START DEVICES
START PORT OSAFE
START IFCONFIG

Commentaires importants
✔ Relation VTAM ↔ TCPIP

PORTNAME OSAFE dans le PROFILE doit correspondre exactement à :
OSATRLE:  PORTNAME=OSAFE
READ/WRITE/DATAPATH doivent être alignés avec Hercules :

1500 QETH iface …
1501 QETH
1502 QETH

