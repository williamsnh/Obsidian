# Résumé — Portage multi-MCU SPI Ethernet (mikroSDK) + bring-up EtherCAT LAN9252

## Contexte / Objectif

Stage : portage d'une librairie Ethernet SPI (trames brutes, sans OS réseau) sur mikroSDK, testée sur plusieurs MCU via socket Sibrain interchangeable (carte UNI-DS v8). Architecture générique : `spi_ethernet.h/.c` (couche commune) + `spi_ethernet_<chip>.c/.h` (bas niveau spécifique) + `main.c` unique multi-puces via macro `SPI_ETH_CHIP`.

## Matériel

- **Carte** : UNI-DS v8 (socket Sibrain)
- **MCU validés** : STM32F429ZIT6, PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L, ATMEGA6450V8U, **PIC18F97J94 (XC8)**
- **Puces SPI Ethernet testées/en cours** :
    - **ENC28J60** (ETH Click) — driver complet et validé, ARP/ICMP/TCP/HTTP fonctionnels
    - **W5500** (ETH Click, MACRAW socket 0) — driver complet et validé sur PIC18F97J94 (ping/HTTP/ICMP OK)
    - **LAN9252** (EtherCAT Click) — **en cours**, bring-up SPI minimal en cours de finalisation (voir ci-dessous), pas encore de logique EtherCAT réelle

## Architecture logicielle

- `spi_ethernet.h` : struct générique `spi_ethernet_t` + table `spi_ethernet_driver_t` (init/send/read/available/link/mac/ip/reset), sélection de puce via `#define SPI_ETH_CHIP` — **doit être défini avant tout `#include`** dans `main.c` (piège de préprocesseur déjà rencontré et corrigé). Constantes `ENC28J60=0`, `W5500=1`, `LAN9252=2` définies dans le header.
- `spi_ethernet_enc28j60.c/.h`, `spi_ethernet_w5500.c/.h`, `spi_ethernet_lan9252.c/.h` : implémentations bas niveau, même structure/conventions.
- `main.c` unique, avec blocs `#if SPI_ETH_CHIP == ...` autour des parties non génériques (PHY ID ENC28J60 only, log ID_REV LAN9252 only, boucle ARP/ICMP/TCP désactivée pour LAN9252).
- Config réseau test : IP `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01` (codées en dur, **identiques sur toutes les cartes** → source de conflits ARP/IP si plusieurs cartes restent branchées, déjà diagnostiqué).

## LAN9252 — état d'avancement détaillé

### Différence fondamentale par rapport à ENC28J60/W5500

Le LAN9252 n'est **pas** un contrôleur MAC/PHY "trames brutes" mais un **ASIC esclave EtherCAT (ESC)** : le MCU ne parle pas Ethernet, il configure des registres CSR (AL Control/Status, Sync Managers, FMMU) ; le traitement EtherCAT (FMMU, process data) se fait dans le silicium à la volée sur une chaîne physique de 2 ports. Nécessite un **maître EtherCAT logiciel côté PC** (SOEM recommandé) pour tout test réel — pas de `ping`/`curl` possible au niveau EtherCAT natif. Un fichier **ESI XML** sera nécessaire pour la reconnaissance par un maître.

### Format SPI confirmé (datasheet DS00001909, §10.2, Table 10-1)

- **READ** = `0x03` (adresse 2 octets MSB-first, pas de dummy, max 30 MHz)
- **WRITE** = `0x02` (adresse 2 octets MSB-first, pas de dummy)
- **FASTREAD** = `0x0B` (1 dummy byte, max 80 MHz, pas encore utilisé)
- Registres **32 bits (DWORD)**, données transmises **LSB first** (contrairement à W5500/ENC28J60, big-endian) — piège à surveiller à chaque nouvelle fonction bas niveau.
- Pas de byte de contrôle bank-select comme W5500 : instruction + adresse suffisent.

### Registres de bring-up confirmés

|Registre|Offset|Rôle|
|---|---|---|
|**BYTE_TEST**|`0x064`|RO, fixe, attendu `0x87654321` — valide que le lien SPI répond|
|**HW_CFG**|`0x074`|bit 27 = READY — attendre avant tout autre accès|
|**ID_REV**|`0x050`|bits 31:16 = Chip ID (attendu `0x9252`), bits 15:0 = révision|

### Fichiers créés : `spi_ethernet_lan9252.c/.h`

- Primitives `lan9252_read_reg32`/`write_reg32` (format ci-dessus)
- `lan9252_wait_ready()` : poll BYTE_TEST puis HW_CFG.READY (timeouts ~500ms chacun)
- `lan9252_get_id_rev()` (32 bits complet) / `lan9252_get_rev()` (octet bas seulement, pour compat nommage `spi_eth_get_rev`)
- Tous les callbacks de `spi_ethernet_driver_t` non applicables (send_packet, read_packet, set_mac, set_ip, get_link_status...) laissés en **stubs commentés**, retournant des valeurs neutres — à réécrire avec la vraie logique EtherCAT plus tard
- **`.reset` volontairement non initialisé** dans le struct `lan9252_driver` (comme ENC28J60/W5500)
- Macro générique ajoutée dans `spi_ethernet.h` : `#define spi_eth_get_rev lan9252_get_rev` (dans le bloc `#if SPI_ETH_CHIP == LAN9252`)

### Bugs XC8/PIC18 rencontrés et résolus sur ce driver

1. **Nom de variable `data`** (paramètre ou tableau local) → mot réservé/qualificateur mémoire implicite sur ce toolchain → provoque "Invalid declarator", "'}' expected ';' found" en cascade. **Déjà rencontré deux fois** (W5500 puis LAN9252) → règle à appliquer systématiquement : ne jamais nommer une variable `data` sur ce compilateur.
2. **Nom de champ struct `.reset`** initialement suspecté comme collision macro (RCON/reset bits PIC18) → retiré du designated initializer par précaution, en cohérence avec ENC28J60/W5500 qui ne l'utilisaient pas non plus.
3. `#define SPI_ETH_CHIP` doit être **avant** tout `#include` dans `main.c` (piège récurrent, déjà rencontré pour W5500 et reproduit à l'identique pour LAN9252).

### À corriger immédiatement (prochaine étape technique)

Le log actuel utilise `spi_eth_get_rev()` (retourne `uint8_t`, donc seulement l'octet bas de la révision) au lieu de `lan9252_get_id_rev()` (32 bits complet, seul moyen de voir `0x9252` attendu). Il faut soit ajouter une macro générique `spi_eth_get_id_rev` → `lan9252_get_id_rev` dans `spi_ethernet.h`, soit garder l'appel direct à `lan9252_get_id_rev()` dans le bloc `#if SPI_ETH_CHIP == LAN9252` de `main.c`.

## Prochaines étapes (après validation ID_REV = 0x9252xxxx)

1. Corriger l'affichage ID_REV complet (32 bits) dans `main.c`
2. Installer/lancer un maître EtherCAT logiciel (**SOEM**, Linux) côté PC de test
3. Récupérer/adapter un fichier **ESI XML** pour le LAN9252 (Microchip AN ou fourni par MikroE pour l'EtherCAT Click)
4. Implémenter la machine à états EtherCAT (AL Control/AL Status : Init → Pre-Op → Safe-Op → Op)
5. Configurer les Sync Managers (mailbox + process data) et éventuellement FMMU

## Git

```
master
* new-feature/spi-ethernet   (branche courante, ENC28J60 + W5500 + LAN9252 en cours)
```

Stratégie : merge simple (pas de rebase, historique traçable).

## Tâche DHCP (toujours en pause)

Remplacer l'IP statique par IP dynamique (DHCPDISCOVER/OFFER/REQUEST/ACK, UDP 67/68). Nécessite parsing + checksum UDP (absent actuellement). Concerne ENC28J60/W5500 uniquement (pas de sens pour LAN9252/EtherCAT natif).