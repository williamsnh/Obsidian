## Résumé du projet

**Contexte** : stage portant sur le développement d'une librairie SPI Ethernet pour mikroSDK (MikroElektronika), avec un objectif de portage multi-MCU (la même librairie doit fonctionner sur différentes familles de microcontrôleurs sans changer l'architecture logicielle).

**Environnement matériel** :

- Carte **UNI-DS v8** (Development Systems), qui utilise un système de **Sibrain Socket** permettant d'interchanger facilement les MCU sans changer de carte de base.
- Module réseau : **ETH Click**, basé sur le chip **ENC28J60** (contrôleur Ethernet SPI), branché sur **mikroBUS 2**.
- Config réseau de test : IP statique `172.20.22.200/24`, MAC `02:DE:AD:BE:EF:01`.

**Architecture logicielle** (stable, ne change pas d'un MCU à l'autre) :

- `spi_ethernet.c/h` — couche générique/portable, expose l'API `spi_ethernet_init/send/receive/get_link_status` via une struct à pointeurs de fonctions (`spi_ethernet_driver_t`)
- `spi_ethernet_enc28j60.c/h` — driver bas niveau spécifique au chip (registres, banks SPI, PHY)
- `main.c` — logique applicative (handlers ARP, ICMP, TCP/HTTP) qui passe uniquement par l'API générique

**Historique des cibles testées** :

1. **STM32F429ZIT6** (GCC ARM) — référence 100% fonctionnelle. ARP, ICMP (ping), TCP/HTTP tous validés (testé via curl et navigateur).
    
2. **PIC24EP512GU814** (mikroC AI for dsPIC) — résolu. Fixes : renommage `data`→`buf` (mot réservé), simplification de `log_cfg_t`, et surtout un blocage total au démarrage causé par les Config Bits (Primary Oscillator/cristal externe configuré alors qu'aucun cristal n'est câblé sur ce socket) → basculé en Internal FRC.
    
3. **dsPIC33FJ256GP710A** (mikroC AI for dsPIC) — résolu. Ici au contraire un vrai cristal 25 MHz est câblé, donc FRC donnait du charabia UART (mismatch baudrate) → repassé en Primary Oscillator/25 MHz. Leçon : chaque carte a sa propre réalité matérielle, pas de règle universelle entre sockets.
    
4. **PIC32MX795F512L** (mikroC AI, cœur MIPS) — résolu. Fausse piste sur un calcul PLL alors que le PLL n'était pas engagé ; réglé en remettant Clock = 8 MHz cohérent avec un FRC simple sans PLL.
    
5. **PIC18F97J94** (mikroC PRO for PIC, 8-bit, quasi-C89) — **mis de côté pour l'instant**. Erreurs de conversion de pointeurs en cascade : ce compilateur utilise des pointeurs génériques `__generic_ptr` sur certaines fonctions SPI, et surtout traite `const` comme un qualificateur de mémoire ROM/Flash (pas juste "lecture seule"), donc caster un pointeur RAM en `const uint8_t *` provoque une erreur. Un internal error du compilateur restait aussi à investiguer (déclarations mi-bloc ou nombre de paramètres de `send_tcp`).
    

**Leçon générale** : chaque famille de compilateur mikroC a ses propres extensions de typage et son propre niveau de conformité C — il faut vérifier au cas par cas via les headers réels plutôt que de réutiliser un fix d'une cible à l'autre.