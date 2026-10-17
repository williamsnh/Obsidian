# Résumé du projet — Driver ENC28J60 mikroSDK

## Contexte

- **Stage** : implémentation d'une librairie SPI Ethernet pour mikroSDK (sujet interne MikroElektronika)
- **Carte** : UNI-DS V8, MCU STM32F429ZIT6
- **Module** : ETH Click (ENC28J60) sur mikroBUS 2
- **Référence** : je me suis appuyé sur un **code bare-metal example fonctionnel** (sans mikroSDK, accès direct aux registres/SPI) comme base logique et comme référence de comportement attendu (timings, séquence d'init, workarounds errata ENC28J60, structure ARP/ICMP/TCP) pour porter la même logique dans l'architecture mikroSDK
- **IP statique de test** : `172.20.22.200/24` — MAC `02:DE:AD:BE:EF:01`

## Architecture retenue

- `spi_ethernet.c/h` — couche générique portable (`spi_ethernet_init/send/receive/get_link_status`, struct `spi_ethernet_driver_t`)
- `spi_ethernet_enc28j60.c/h` — driver bas niveau spécifique au chip (registres, banks, PHY, TX/RX bruts)
- `main.c` — logique applicative (ARP, ICMP, TCP, HTTP) via handlers séparés, n'utilisant que l'API générique `spi_ethernet_*`
- Règle validée avec le sujet : séparer transport / logique chip / API utilisateur "where possible" — diagnostics d'init très spécifiques au chip (EREVID, PHY ID) restent acceptables en dehors de l'abstraction stricte

## Commits réalisés

**Commit 1** — SPI init + soft reset + lecture EREVID (0x06) OK

**Commit 2** — Config MAC + lien PHY (PHSTAT2) validés en UART

**Commit 3** — TX OK, trame EtherType custom `0x88B5` envoyée et vérifiée

**Commit 4** — RX OK, réception de trames

**Commit 5** — ARP reply fonctionnel (validé avec `ip neigh`, comportement STALE normal)

**Commit 6** — ICMP Echo Reply OK — ping validé : 0% packet loss, ~7ms RTT

**Commit 7** — Serveur HTTP OK — TCP handshake (SYN/SYN-ACK/ACK) + réponse HTTP 200 + FIN, testé via `curl`/navigateur

**Commit 8** — Nettoyage complet :

- Suppression de tout le code mort (ancien `read_packet` commenté, ARP cache non utilisé, unions/variables DHCP/DNS/gateway jamais exploitées, macros GET_LOW/HIGH_BYTE, prototypes `beta_enc28j60_*` fantômes, struct `enc28j60_cfg_t` inutilisée)
- Fix bug PHY : `enc28j60_phy_write` utilisait la mauvaise bank pour lire `MISTAT` (bank 2 au lieu de 3) — corrigé
- `enc28j60_init()` refactorisé en sous-fonctions claires (`hw_reset`, `sw_reset`, `init_rx_buffer`, `init_tx_buffer`, `init_mac`, etc.)
- `set_mac/get_mac/set_ip/get_ip` implémentés (plus de stubs vides)
- `main.c` : remplacement de `enc28j60_get_link_status()` par le wrapper générique `spi_ethernet_get_link_status(&eth)`
- Codes de retour cohérents ajoutés sur les fonctions MAC/IP

## Points restants identifiés (sujet du stage)

- **DHCP** : pas encore implémenté — discussion en cours pour un futur commit 9 (nécessite UDP + state machine DHCP DISCOVER/OFFER/REQUEST/ACK, IP dynamique `0.0.0.0` → assignée, timeout/retry, fallback IP statique à décider)
- **UDP** : aucun scénario testé — à faire ou à documenter explicitement comme hors scope
- Validation matérielle avec logic analyzer/oscilloscope — pas encore faite
- Test link down/up en cours de fonctionnement (câble débranché/rebranché) — pas encore testé
- Documentation finale (Doxygen, README, exemples séparés NECTO, rapport de validation) — pas commencée
- Gestion d'erreur au niveau `spi_ethernet_init()` (actuellement `void`, ne remonte aucune erreur d'init)

## Fichiers clés actuels

- `main.c` : handlers `handle_arp`, `handle_icmp`, `handle_tcp`, `handle_ip` + boucle principale
- `spi_ethernet_enc28j60.c/h` : driver nettoyé, fonctions d'init découpées
- `spi_ethernet.c/h` : couche d'abstraction générique inchangée depuis le début