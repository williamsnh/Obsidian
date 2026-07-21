# Résumé — Portage SPI Ethernet mikroSDK sur PIC18F97J94 (XC8)

## Contexte projet

Stage : portage multi-MCU d'une lib SPI Ethernet (ENC28J60, chip ETH Click) pour mikroSDK. Carte UNI-DS v8 (socket Sibrain interchangeable). Architecture : `spi_ethernet.c/h` (générique, driver via pointeurs de fonction) + `spi_ethernet_enc28j60.c/h` (bas niveau) + `main.c` (appli test : ARP/ICMP/TCP-HTTP). Config réseau test : IP `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01`.

## Cibles précédentes (toutes résolues)

STM32F429ZIT6 (référence OK), PIC24EP512GU814 (fix: FRC au lieu de cristal externe absent), dsPIC33FJ256GP710A (fix: inverse, Primary Osc 25MHz car cristal réel présent), PIC32MX795F512L (fix: Clock 8MHz sans PLL).

## Cible actuelle : PIC18F97J94, compilateur **XC8** (mikroC PRO for PIC fonctionne parfaitement sur ce même hardware, sans aucune modif)

## Problèmes résolus dans cette session

1. **Erreur linker `(1250) could not find space for variable`** puis **`(1347) psect bssBIGRAM`** : cause = objets RAM >256 bytes forcés dans la classe BIGRAM (contiguë), fragmentée par PIC18 (banks de 256B). Fix appliqué : suppression des fonctions inutilisées `ethernet_send_frame`/`ethernet_receive_frame` dans `spi_ethernet.c` (qui contenaient chacune un buffer statique 1514 bytes, liées de force car même fichier objet que `spi_ethernet_init`). Buffers réduits (`rx_buf`, `tx_pkt` etc). **→ Compile et flash OK maintenant.**
2. Warning résiduel non bloquant : `Unknown argument: '-msummary=mem'` (flag résiduel, probablement d'un autre compilateur, à nettoyer dans les options de projet XC8 mais n'empêche pas la compilation).

## Problème actuel non résolu : UART affiche du charabia (garbage), ping/curl ne passent pas

- Config Bits vérifiés : HS Oscillator + PLL x4 = Clock 64MHz (nécessite cristal 16MHz réel sur le socket — jamais confirmé physiquement, mais probablement non-coupable car mikroC fonctionne sans y toucher).
- **Test décisif fait** : le démo UART minimal du SDK (`uart_open`/`uart_write` bruts, sans passer par `log`) **fonctionne parfaitement** sur XC8 avec les mêmes pins (`USB_UART_TX/RX`), baudrate 115200, minicom `-D /dev/ttyUSB1 -b 115200`.
- **Dans le projet spi_ethernet** : même en remplaçant `log_printf` par un `uart_write` brut avec chaîne fixe via `logger.uart`, ou même en créant un `uart_t` totalement séparé façon démo → **toujours du charabia**.
- `log_init()` (source vérifiée, fichier `log.c` du SDK) est structurellement correct : ring buffers assignés avant `uart_open()`, même séquence que le démo qui marche.
- Passer `log_cfg.is_interrupt = false` après `LOG_MAP_USB_UART` → aucun changement.
- **Test LED multi-checkpoints fait** (5 GPIO PA0-PA4, une par étape clé) : confirme que `main()` s'exécute intégralement jusqu'à la boucle `while(1)` sans crash ni blocage (mais test à refaire proprement avec reset `low` + flash bref par étape, car actuellement les LEDs restent toutes allumées sans distinction claire de séquence — pin3 dupliquée sur 2 checkpoints différents).

## Hypothèses non encore tranchées

1. **Jamais testé/comparé** : Config Bits du projet `UART demo` (qui marche) vs projet `spi_ethernet` (qui ne marche pas) — capture Config Bits `spi_ethernet` déjà fournie (Clock 64MHz, HS+PLLx4, voir détail si besoin), celle du démo UART jamais vérifiée. **C'est le test prioritaire restant.**
2. Possible conflit de pins physique entre UART et SPI/ENC28J60 sur ce mikroBUS/board précis (jamais vérifié).
3. Possible effet de bord des appels indirects via pointeurs de fonction (`eth->drv->send_packet(...)` etc.) sur l'optimiseur mémoire non-reentrant de XC8/PIC18 — hypothèse non totalement écartée mais partiellement affaiblie (le crash total étant exclu par le test LED).

## Prochaine étape recommandée

1. Refaire le test LED proprement (reset low + flash bref distinct par checkpoint) pour confirmer précisément jusqu'où le code va normalement.
2. **Comparer Config Bits `UART demo` vs `spi_ethernet`** — jamais fait, priorité absolue.
3. Si Config Bits identiques : chercher conflit de pins/wiring physique entre SPI (ENC28J60) et UART sur ce socket.