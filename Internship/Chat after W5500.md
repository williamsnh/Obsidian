# Résumé — Projet de stage : Portage multi-MCU d'une lib SPI Ethernet (mikroSDK)

## Contexte / Objectif

Portage d'une librairie Ethernet SPI (raw frames, sans OS réseau) sur mikroSDK, testée successivement sur plusieurs MCU via le socket interchangeable Sibrain (carte UNI-DS v8). Tests validés : ARP reply, ICMP echo (ping), mini serveur HTTP TCP raw (port 80).

## Matériel

- **Carte** : UNI-DS v8 (socket Sibrain)
- **MCU portés et validés** : STM32F429ZIT6, PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L, ATMEGA6450V8U, **PIC18F97J94 (XC8)** — tous fonctionnels
- **Puces Ethernet SPI testées** :
    - **ENC28J60** (ETH Click) — driver de référence, complet (ARP/ICMP/TCP/HTTP validés)
    - **W5500** (ETH Click, mode MACRAW socket 0) — driver ajouté récemment, validé fonctionnel sur PIC18F97J94 (ping/HTTP/ICMP OK)
- **Prochain click à porter** : **EtherCAT Click, puce LAN9252** (protocole EtherCAT, architecture différente : 2 ports Ethernet physiques + interface SPI vers un ASIC EtherCAT, pas un simple contrôleur MAC/PHY raw comme ENC28J60/W5500)

## Architecture logicielle

- `spi_ethernet.h/.c` : couche générique, struct `spi_ethernet_t` + table de fonctions `spi_ethernet_driver_t` (init/send/read/available/link/mac/ip), sélection de puce via macro `SPI_ETH_CHIP` (`ENC28J60`/`W5500`) **définie avant tout `#include`** dans `main.c` (piège de préprocesseur déjà rencontré et corrigé)
- `spi_ethernet_enc28j60.c/.h` et `spi_ethernet_w5500.c/.h` : implémentations bas niveau spécifiques, même structure/conventions pour éviter les erreurs de portage
- `main.c` : init SPI/NIC, lecture révision puce (+ PHY ID si ENC28J60 uniquement — le W5500 n'expose pas ce registre), boucle `handle_arp` / `handle_icmp` / `handle_tcp` (dispatch via `handle_ip`), mini serveur HTTP TCP raw (`send_tcp`, `ip_checksum`, `tcp_checksum`)
- Config réseau test actuelle : IP statique `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01` (**codées en dur, identiques sur toutes les cartes** — source de conflits ARP/IP entre cartes de dev restées branchées simultanément, déjà rencontré et diagnostiqué)

## Bugs résolus (classés par cause)

1. **`log_printf` formats combinés** (`%X%X`, `%s` dynamique) → comportement incohérent selon MCU (décalage `va_list` PIC18, padding ATmega, `%c` garbage PIC32). Règle adoptée : un seul `%X`/argument par appel.
2. **Garbage UART PIC32MX795** → mismatch horloge système (PBCLK/BRG), pas logiciel — vérifier fuses `FNOSC/FPLLIDIV/FPLLMUL/FPLLODIV/FPBDIV`.
3. **Portage W5500, erreurs de compilation XC8** :
    - Initialiseurs de tableaux composés (`{...}`) inline → remplacés par affectations champ par champ
    - Nom de paramètre `data` → mot réservé/qualificateur mémoire implicite sur ce toolchain → renommé
    - Macro `SPI_ETH_CHIP` définie après `#include "spi_ethernet.h"` dans `main.c` → figeait le driver par défaut (ENC28J60) quel que soit le define voulu → corrigé en plaçant le `#define` avant tout `#include`
    - `spi_eth_phy_read`/`PHHID1` non applicable au W5500 (pas de registre d'ID PHY exposé par le WIZnet) → code entouré d'un `#if SPI_ETH_CHIP == ENC28J60`
4. **Duplicatas ping (`DUP!`, deux TTL différents)** → pas un bug logiciel : conflit MAC+IP codés en dur, partagés entre plusieurs cartes de dev restées alimentées simultanément sur le réseau de test. Diagnostiqué via TTL (64=Linux/firmware embarqué vs 128=autre appareil probablement Windows) et temps de réponse (~6ms firmware SPI vs ~0.12ms autre machine). Solution retenue : MAC unique par carte pour la suite du stage.

## Organisation Git

```
master
* new-feature/spi-ethernet   (branche courante, tout le travail SPI Ethernet, ENC28J60 + W5500)
```

Stratégie : merge simple (pas de rebase, historique traçable). Branches futures créées à partir de `new-feature/spi-ethernet` selon les besoins de test (chip par chip).

## Tâche DHCP (mise en pause)

Objectif futur : remplacer l'IP statique par une IP obtenue dynamiquement (DHCPDISCOVER/OFFER/REQUEST/ACK, ports UDP 67/68). Nécessite gestion UDP (parsing + checksum, absente actuellement — seuls ARP/ICMP/TCP gérés). mDNS envisagé après.

## Prochaine étape immédiate

Porter le driver pour l'**EtherCAT Click (LAN9252)** en suivant la même structure de fichiers (`spi_ethernet_lan9252.c/.h`) que ENC28J60/W5500, en anticipant que LAN9252 est un ASIC EtherCAT (pas un simple MAC/PHY raw) — l'API haut niveau (`spi_ethernet_driver_t`) devra probablement être adaptée ou étendue pour gérer les spécificités EtherCAT (registres CSR, FMMU, PDO, etc.) plutôt que du simple envoi/réception de trames Ethernet brutes.