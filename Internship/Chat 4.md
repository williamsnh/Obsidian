## Résumé — Portage librairie SPI Ethernet mikroSDK multi-MCU

### Contexte projet

**Stage** : librairie SPI Ethernet pour mikroSDK (MikroElektronika), portage multi-MCU. Carte **UNI-DS v8** (Development Systems, Sibrain Socket) avec MCU interchangeables. Module réseau : **ETH Click (ENC28J60)** sur mikroBUS 2. IP test statique `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01`.

### Architecture logicielle (stable, ne change jamais)

- `spi_ethernet.c/h` — couche générique portable (API `spi_ethernet_init/send/receive/get_link_status`, struct `spi_ethernet_driver_t` avec pointeurs de fonctions)
- `spi_ethernet_enc28j60.c/h` — driver bas niveau ENC28J60 (registres, banks SPI, PHY)
- `main.c` — logique applicative (handlers ARP, ICMP, TCP/HTTP) via l'API générique uniquement

### Historique des cibles testées (dans l'ordre)

**1. STM32F429ZIT6 (GCC ARM)** — référence fonctionnelle à 100%. ARP/ICMP/TCP-HTTP tous validés (ping OK, serveur HTTP testé via curl/navigateur).

**2. PIC24EP512GU814 (compilateur mikroC AI for dsPIC)**

- Fix : renommer tous les paramètres `data` → `buf` (mot réservé XC16/mikroC — ne s'applique en fait qu'à certains compilateurs Microchip)
- Fix : `log_cfg.is_interrupt = 0;` supprimé — `log_cfg_t` sur cette implémentation n'a qu'un champ `level`
- Blocage total au démarrage (curseur figé sur 1ère ligne disassembly) → cause : **Config Bits, `Initial Oscillator Source Selection = Primary Oscillator (HS Crystal)` alors qu'aucun cristal externe n'est câblé** sur ce socket. Fix : basculer sur **"Internal Fast RC (FRC)"**. Watchdog déjà désactivé (pas la cause).
- **Résolu et fonctionnel.**

**3. dsPIC33FJ256GP710A (mikroC AI for dsPIC)**

- Repart de la config PIC24, teste FRC → texte UART illisible (charabia) → mismatch baudrate car FRC réel (~7,37 MHz) ≠ champ Clock affiché
- Repasse en **Primary Oscillator, Clock = 25 MHz** → texte lisible → confirmé qu'un vrai cristal 25 MHz est câblé sur cette carte, contrairement au PIC24. **Résolu.**
- Leçon retenue : chaque carte a sa propre réalité matérielle (cristal présent ou non), pas de règle universelle.

**4. PIC32MX795F512L (mikroC AI, cœur MIPS)**

- Clock affiché 8 MHz, `Oscillator Selection Bits = Fast RC Osc (FRC)` (sans PLL) → texte illisible malgré tout
- Fausse piste initiale : calcul PLL (÷4 ×20 = 40MHz) alors que le PLL n'était pas engagé (mode FRC simple, pas "FRC w/ PLL")
- **Résolu** en remettant simplement Clock = 8 MHz (cohérent avec FRC sans PLL réellement actif).

**5. PIC18F97J94 (compilateur mikroC PRO for PIC — 8-bit, très strict, quasi C89)**

- Erreurs "Illegal pointer conversion" en cascade. Cause : ce compilateur utilise un système de pointeurs génériques `__generic_ptr` (visible dans `drv_spi_master.h` : `spi_master_write` attend `uint8_t * __generic_ptr`, alors que `spi_master_write_then_read`/`spi_master_read` attendent des pointeurs classiques `uint8_t*` — **distinction cruciale, pas une règle uniforme**)
- Erreur similaire déjà rencontrée avec `log_printf` (`const code char * __generic_ptr`)
- Fix appliqué : cast `(uint8_t * __generic_ptr)` ajouté sur tous les appels `spi_master_write` dans `spi_ethernet_enc28j60.c`, **mais pas** sur `spi_master_write_then_read` (retiré après une erreur de surcorrection)
- **Nouveau problème découvert, encore en cours** : ce compilateur exige aussi un **cast explicite `(const uint8_t *)`** partout où un buffer non-const local (tableaux `uint8_t[]`, `&variable`) ou `NULL` est passé à un paramètre `const uint8_t *` (ex: `spi_ethernet_send`, `send_tcp` avec son paramètre `payload`). ~17 erreurs de ce type actuellement dans `main.c`, `spi_ethernet.c` (`ethernet_send_frame`), `spi_ethernet_enc28j60.c` (`enc28j60_send_packet`).
- **Correctifs donnés, pas encore confirmés appliqués/recompilés** :
    - `spi_ethernet.c` → `ethernet_send_frame` : `return spi_ethernet_send( eth, ( const uint8_t * )buffer, pos );`
    - `spi_ethernet_enc28j60.c` → `enc28j60_send_packet` : `enc28j60_write_mem( ( const uint8_t * )&ctrl, 1 );`
    - `main.c` → tous les appels `spi_ethernet_send(...)` avec buffer local (`pkt`, `reply`, `tx_buf`) et tous les `NULL` passés à `send_tcp(...)` → caster en `(const uint8_t *)`
- **Erreur additionnelle non résolue** : "Internal error ''" (crash interne du compilateur) à la fin de `handle_tcp()`. Hypothèse principale : déclarations de variables (`uint32_t new_ack`, `uint16_t resp_len`) **au milieu d'un bloc** `{}`, non supporté par ce compilateur quasi-C89 → il faut remonter toutes les déclarations en tout début de chaque bloc `{}`. Correctif proposé mais pas encore testé.

### Prochaine étape immédiate

1. Appliquer tous les casts `(const uint8_t *)` listés ci-dessus dans `main.c`
2. Restructurer `handle_tcp()` pour mettre toutes les déclarations de variables en début de bloc (style C89)
3. Recompiler et voir si les erreurs restantes (17 pointer conversions + internal error) disparaissent, ou si l'internal error persiste (auquel cas investiguer une autre piste, ex: limite du compilateur sur le nombre de paramètres de `send_tcp` qui en a 9)

### Leçon générale retenue sur ce portage multi-MCU

Chaque famille de compilateur mikroC (AI for dsPIC vs PRO for PIC 8-bit) a ses propres extensions/contraintes de typage de pointeurs (`__generic_ptr`, conversions const strictes) et son propre niveau de conformité C (C89 vs plus permissif) — il faut vérifier au cas par cas via les prototypes réels dans les headers générés (`drv_spi_master.h`, `log.h`) plutôt que de supposer qu'un fix qui a marché sur une cible s'applique identiquement à la suivante.