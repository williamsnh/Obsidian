# Résumé — Stage portage librairie SPI Ethernet (mikroSDK)

## Contexte / objectif

Stage : porter une librairie Ethernet SPI (trames brutes, sans OS réseau) sur mikroSDK, testée sur plusieurs MCU via socket Sibrain interchangeable (carte UNI-DS v8). Architecture générique :

- `spi_ethernet.h/.c` — couche commune, struct `spi_ethernet_driver_t` (callbacks init/send_packet/read_packet/available/get_link_status/set_mac/get_mac/set_ip/get_ip)
- `spi_ethernet_<chip>.c/.h` — un driver par puce
- `main.c` unique multi-puces via macro `SPI_ETH_CHIP`

## Matériel / toolchain

- MCU principal de test : **STM32F429ZIT6** sur carte **UNI-DS v8** (socket Sibrain)
- Compilateur : **mikroC AI for ARM**
- Autres MCU validés (même socket) : PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L, ATMEGA6450V8U, PIC18F97J94
- Piège compilateur récurrent : ne jamais nommer une variable `data` → toujours `val`/`value`

## Avancement puces SPI Ethernet (contrôleurs MAC/PHY "trames brutes", tous compatibles `spi_ethernet_driver_t`)

1. ✅ **ENC28J60** (ETH Click) — driver complet et validé, ARP/ICMP/TCP/HTTP fonctionnels
2. ✅ **W5500** (ETH Click, socket 0 MACRAW) — driver complet et validé, ping/HTTP/ICMP OK. Fix appliqué : `w5500_get_link_status()` vérifie désormais `VERSIONR == 0x04` avant de faire confiance à PHYCFGR (évite faux "LINK UP" quand rien n'est branché ou mauvais mapping mikroBUS)
3. ✅ **LAN9250** (ETH3 Click) — driver complet (FIFO TX/RX, SPI READ=0x03/WRITE=0x02, données 32 bits LSB-first, registres indirects MAC_CSR/MII), intégré et fonctionnel
4. 🔄 **LAN9252** (EtherCAT Click) — en cours. Pas un contrôleur MAC/PHY trames brutes mais un ASIC esclave EtherCAT (ESC), registres CSR (AL Control/Status, Sync Managers, FMMU), nécessite maître EtherCAT logiciel (SOEM) + fichier ESI XML. Bring-up SPI minimal (BYTE_TEST, HW_CFG READY, ID_REV=0x9252xxxx) en finalisation
5. 🔄 **W6100** — **chip actuellement en cours de travail** (voir ci-dessous)

## Cas particulier hors abstraction SPI : ETH WIZ 4 Click / WIZ-IP20

Board reçu par erreur d'identification initiale (confondu avec IP20/IP55). Analyse terminée et actée :

- **WIZ-IP20** = module UART-vers-Ethernet autonome (ARM Cortex-M3, firmware WIZnet intégré, pile TCP/IP complète _dans_ le module). Communique en **UART** (RXD/TXD, pas de SPI du tout — confirmé par schéma `ETH_WIZ_4_Click_v100_Schematic.PDF` et par la datasheet `WIZ-IP20_User_Manual_V1_0`). USB-C du board sert uniquement au flash firmware du module, pas à la donnée applicative.
- Décision architecturale actée : **ne pas** forcer ce chip dans `spi_ethernet_driver_t`. Driver séparé `spi_ethernet_wizip20.c/.h` (types `wizip20_t`/`wizip20_cfg_t`, fonctions `wizip20_configure/hw_reset/uart_write/uart_read`), piloté par commandes AT (syntaxe exacte encore à confirmer — tableaux AT du manuel pages 22-24 non extraits).
- Dans `spi_ethernet.h` : `#define WIZ_IP20 3` ajouté à la liste des chips, mais la branche `#elif SPI_ETH_CHIP == WIZ_IP20` dans `spi_ethernet.h` reste **vide** (pas de `spi_eth_cfg_t`, pas d'alias driver — juste l'include du header dédié), car ce n'est pas un contrôleur SPI.
- Dans `main.c` : séparation via `#if SPI_ETH_CHIP == WIZ_IP20 ... #else ... #endif` à plusieurs endroits (déclarations globales, init, boucle principale) — le code ARP/ICMP/TCP handlers reste inchangé mais n'est utilisé que pour les vrais chips SPI.
- Étape de dev actuelle validée avec l'utilisateur : approche incrémentale (UART OK → reset OK → AT/OK → version → IP → TCP → data), en commençant par un test minimal `wizip20_uart_write("AT\r\n")` + lecture brute de la réponse, sans aucun parsing AT pour l'instant.
- Bug de compilation en cours de résolution : `pin_name_t` non reconnu dans `spi_ethernet_wizip20.h` → fix = ajouter `#include "drv_spi_master.h"` (c'est ce header qui définit `pin_name_t`, même si le chip n'utilise pas le SPI physiquement).
- Convention actée : fichiers nommés `spi_ethernet_wizip20.c/.h` (pas de nouveau sous-dossier, tout à plat comme les autres drivers).

## Sujet actuel de la conversation

L'utilisateur passe maintenant au chip **W6100** (déjà présent en `#elif` dans `spi_ethernet.h` d'après un extrait de code fourni, mais son driver `.c/.h` n'a pas encore été discuté/écrit dans cette conversation). Aucune datasheet W6100 n'a encore été fournie ni analysée à ce stade.