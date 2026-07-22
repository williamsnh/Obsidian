## Résumé du contexte — Portage SPI Ethernet mikroSDK sur PIC18F97J94 (XC8)

**Matériel/Projet :**

- Stage : portage multi-MCU d'une lib SPI Ethernet (chip ENC28J60, ETH Click) sur mikroSDK
- Carte : UNI-DS v8 (socket Sibrain interchangeable), MCU actuel : PIC18F97J94, compilateur XC8
- Architecture : `spi_ethernet.c/h` (générique) + `spi_ethernet_enc28j60.c/h` (bas niveau) + `main.c` (test ARP/ICMP/TCP-HTTP)
- Config réseau test : IP `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01`
- UART : pins `USB_UART_TX/RX` = `GPIO_PC6/PC7` ; mikroBUS 1 (SPI/ENC28J60) = `PL1, PD4-PD6, PJ4, PA3, PG4, PB0, PE0/E1, PC3/PC4` → **pas de conflit de pins identifié**

**Cibles précédentes (toutes résolues, hors sujet actuel) :** STM32F429ZIT6, PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L — tous OK.

**Historique récent sur PIC18F97J94 :**

1. Bug linker RAM (BIGRAM/bank PIC18) → résolu en supprimant du code mort.
2. Compile et flash OK.

**Problème actuel non résolu :**

- **UART bas niveau (`uart_open`/`uart_write` bruts, sans passer par `log`) fonctionne parfaitement** dans le projet principal, sous XC8, sur ce PIC18F97J94.
- **`log_init` + `log_printf` (couche `log` du SDK MikroE) ne fonctionne pas** : `log_init()` retourne bien `LOG_SUCCESS`, mais aucune donnée ne sort physiquement sur l'UART — même en bypassant `log_printf` et en appelant `uart_write()` directement sur `logger.uart` juste après `log_init`. Testé avec `is_interrupt = true` et `false` (`LOG_MAP_USB_UART` force `true`, override testé après la macro) : aucun changement, toujours rien.
- Pistes déjà écartées : conflit de pins UART/SPI (vérifié via `mcu_card.h`/`board.h`), mode interrupt vs polling, structure de `log_cfg_t`/`log_init` (code source de `log.c` vérifié, semble correct), qualificatifs `code`/`__generic_ptr` dans `log_printf_implementation.c` (possible mais non confirmé, et de toute façon le bypass direct via `uart_write` échoue aussi donc ce n'est pas isolé à `log_printf`).
- Piste non encore explorée : **`preinit()`** appelé en tout début de `main()` avant `log_init` — pourrait toucher à une config bas niveau (registres analogiques/digitaux sur PC6/PC7 ou autre) qui casserait la sortie UART physique même si `log_init` réussit. Pas encore vérifié si le petit projet UART minimal (qui marche) appelle aussi `preinit()`.

**Fichiers déjà fournis dans la conversation précédente :** `log.h`, `log.c`, `log_printf_implementation.h/.c`, `board.h`, `mcu_card.h`, et les deux versions de `main.c` (avec `log_printf` qui échoue, et avec `UART_LOG`/`uart_write` qui fonctionne).