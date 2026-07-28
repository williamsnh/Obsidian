## Résumé du contexte — Portage SPI Ethernet mikroSDK (multi-MCU) + démarrage DHCP

**Projet / Stage :** Portage multi-MCU d'une lib SPI Ethernet (chip ENC28J60, ETH Click) sur mikroSDK.

**Matériel :**

- Carte : UNI-DS v8 (socket Sibrain interchangeable)
- MCU en cours : PIC18F97J94, compilateur XC8
- MCU déjà portés et validés : STM32F429ZIT6, PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L, ATMEGA6450V8U
- Architecture code : `spi_ethernet.c/h` (générique) + `spi_ethernet_enc28j60.c/h` (bas niveau) + `main.c` (test ARP/ICMP/TCP-HTTP)
- Config réseau test actuelle : IP statique `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01`
- UART test/log (PIC18F97J94) : pins `USB_UART_TX/RX` = `GPIO_PC6/PC7` ; mikroBUS 1 (SPI/ENC28J60) = `PL1, PD4-PD6, PJ4, PA3, PG4, PB0, PE0/E1, PC3/PC4`

**État du code (`main.c`) — fonctionnel sur tous les MCU validés :**

- Init SPI/NIC, lecture EREVID/PHHID1, attente link
- `handle_arp` (réponse ARP), `handle_icmp` (ping), `handle_tcp` (mini serveur HTTP TCP raw : `send_tcp`, `ip_checksum`, `tcp_checksum`)
- Dispatch dans `handle_ip` (ICMP protocole 1, TCP protocole 6)
- Test validé : `ping` + `curl -v http://172.20.22.200/` répondent correctement

**Bugs résolus récemment (log/format) :**

- `log_printf` avec formats combinés (`%X%X`, `%s` dynamique) → comportement incohérent selon MCU (décalage `va_list` sur PIC18, padding variable sur ATmega, `%c` garbage sur PIC32). **Solution retenue : un seul `%X` par appel `log_printf`, un seul argument à la fois, jamais de format combiné.**
- Garbage UART total sur PIC32MX795 identifié comme un mismatch **horloge système (PBCLK/BRG)**, pas un bug logiciel — vérifier fuses `FNOSC/FPLLIDIV/FPLLMUL/FPLLODIV/FPBDIV`.

**Organisation Git actuelle :**

```
master
* new-feature/spi-ethernet   (branche courante, tout le travail SPI Ethernet)
```

- Prochaine étape : créer `feature/dhcp-client` à partir de `new-feature/spi-ethernet`
- Stratégie de réintégration choisie : **merge simple** (pas de rebase, pour garder l'historique traçable — conseillé pour le stage)
- Raison de la branche séparée : possibilité de devoir interrompre le DHCP pour tester d'autres chips que l'ENC28J60 sur l'ETH Click

**Prochaine tâche : implémenter un client DHCP**

- Objectif : remplacer l'IP statique codée en dur par une IP obtenue dynamiquement auprès du serveur DHCP du réseau local (typiquement le routeur), pas lié à la machine du testeur
- Nécessite : gestion UDP (parsing + checksum, actuellement absente — seuls ARP/ICMP/TCP sont gérés)
- Séquence standard : `DHCPDISCOVER` (broadcast) → `DHCPOFFER` → `DHCPREQUEST` → `DHCPACK`
- Ports UDP : 67 (serveur), 68 (client)
- Une fois l'IP obtenue, l'afficher via `log_printf` (format `%X` unique par appel, cf. bugs résolus) pour que l'utilisateur sache quelle adresse utiliser pour `ping`/`curl`
- mDNS (`http://xxx.local`) envisagé mais reporté après le DHCP

**Fichiers disponibles si besoin :** `main.c` complet (version ARP/ICMP/TCP), `log.c/h`, `log_printf_implementation.c/h`, `board.h`, `mcu_card.h`.