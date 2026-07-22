## Résumé du contexte — Portage SPI Ethernet mikroSDK (PIC18F97J94 + multi-MCU)

**Projet / Stage :** Portage multi-MCU d'une lib SPI Ethernet (chip ENC28J60, ETH Click) sur mikroSDK.

**Matériel :**

- Carte : UNI-DS v8 (socket Sibrain interchangeable)
- MCU actuel en cours : PIC18F97J94, compilateur XC8
- MCU déjà portés et validés : STM32F429ZIT6, PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L, ATMEGA6450V8U (tous OK, hors bugs de log ponctuels)
- Architecture code : `spi_ethernet.c/h` (générique) + `spi_ethernet_enc28j60.c/h` (bas niveau) + `main.c` (test ARP/ICMP/TCP-HTTP)
- Config réseau test actuelle : IP statique `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01`
- UART test/log : pins `USB_UART_TX/RX` = `GPIO_PC6/PC7` (sur PIC18F97J94) ; mikroBUS 1 (SPI/ENC28J60) = `PL1, PD4-PD6, PJ4, PA3, PG4, PB0, PE0/E1, PC3/PC4`

**État actuel du code (`main.c`) :**

- Fonctionnel : init SPI/NIC, lecture EREVID/PHHID1, attente link, `handle_arp` (réponse ARP), `handle_icmp` (ping), `handle_tcp` (mini serveur HTTP via TCP raw avec `send_tcp`, `ip_checksum`, `tcp_checksum`), dispatch dans `handle_ip` (ICMP protocole 1, TCP protocole 6).
- Test validé : ping + `curl -v http://172.20.22.200/` répondent correctement sur PIC32, ATmega, etc.

**Bugs résolus récemment (log/format, PIC18) :**

- `log_printf` avec formats combinés (`%X%X`, `%s` dynamique) donne des résultats incohérents selon le MCU (décalage `va_list`, padding variable, `%c` garbage sur PIC32). Solution retenue : **un seul `%X` par appel `log_printf`, un seul argument à la fois**, jamais de format combiné multi-arguments.
- Un bug d'affichage garbage total sur PIC32MX795 (`ping`/UART illisible même sur du texte simple) a été identifié comme un problème de **configuration horloge système (PBCLK/BRG mismatch)**, pas un bug logiciel — à vérifier via les fuses `FNOSC/FPLLIDIV/FPLLMUL/FPLLODIV/FPBDIV`.

**Prochaine tâche : implémenter un client DHCP**

- Priorisé avant mDNS (qui était envisagé pour remplacer l'IP statique par un nom `http://xxx.local`, mais reporté).
- Nécessite : gestion UDP (parsing + checksum, actuellement absente du code — seuls ARP/ICMP/TCP sont gérés), séquence DHCP standard DISCOVER → OFFER → REQUEST → ACK, remplacement de l'IP statique codée en dur par l'IP obtenue dynamiquement.
- Protocole DHCP utilise UDP ports 67 (serveur) / 68 (client), broadcast MAC/IP pour DISCOVER/REQUEST.

**Fichiers disponibles si besoin :** `main.c` complet (version actuelle avec ARP/ICMP/TCP), `log.c/h`, `log_printf_implementation.c/h`, `board.h`, `mcu_card.h`.