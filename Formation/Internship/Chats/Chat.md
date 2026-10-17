## Résumé du projet — ENC28J60 driver mikroSDK

### Contexte

- **Carte** : UNI-DS V8, MCU STM32F429ZIT6
- **Module** : ETH Click (ENC28J60) sur **mikroBUS 2**
- **Objectif** : implémenter un driver SPI Ethernet dans mikroSDK
- **Connexion** : ETH Click → routeur → PC (IP PC : 172.20.22.117/24)

### Pins mikroBUS 2

|Signal|Pin|
|---|---|
|SCK|GPIO_PA5|
|MISO|GPIO_PA6|
|MOSI|GPIO_PB5|
|CS|GPIO_PB2|
|RST|GPIO_PE12|

### Architecture du code

Trois fichiers principaux :

- `main.c` — init UART, SPI, CS/RST, contexte eth, appel `spi_ethernet_init()`
- `spi_ethernet_enc28j60.c` — driver ENC28J60
- `spi_ethernet_enc28j60.h` — déclarations

### Ce qui est fait et commité

- SPI init OK
- Reset matériel + soft reset OK
- Lecture EREVID = **0x06** (révision silicium valide) ✅
- `enc28j60_get_rev()` ajoutée pour exposer `enc_hwRev`

### Corrections importantes déjà appliquées

1. **CS en GPIO manuel** — `spi_master_select_device()` ne fonctionne pas avec un `digital_out_t`, remplacé partout par `digital_out_low/high(&current_eth->cs)`
2. **Fix `select_bank()`** — l'ancienne version faisait `enc28j60_write_reg(ECON1, bank)` qui écrasait tout le registre ; corrigé avec `enc28j60_set_bit_reg(ECON1, bank & 0x03)`

### État actuel du `spi_ethernet_enc28j60.c`

- `current_eth` = pointeur statique global vers le contexte `spi_ethernet_t`, initialisé dans `enc28j60_init()` et mis à jour dans `send_packet` / `read_packet`
- Toutes les fonctions SPI bas-niveau (`read_reg`, `write_reg`, `read_mem`, `write_mem`, `set_bit_reg`, `clear_bit_reg`, `soft_reset`) utilisent `digital_out_low/high(&current_eth->cs)` pour le CS
- `enc28j60_get_rev()` retourne `enc_hwRev` (variable statique remplie dans `init()`)
- Plusieurs fonctions encore vides (TODO) : `get_link_status`, `set_mac`, `get_mac`, `set_ip`, `get_ip`

### Prochaine étape (commit 2)

Valider la configuration Ethernet :

- Relire MACON1, MACON3, MAADRx après `enc28j60_init()` pour vérifier que les registres sont bien écrits
- Lire `PHSTAT2` pour confirmer que le lien PHY est détecté (câble branché)
- Logger tout ça en UART
- Si OK → commit "enc28j60: MAC config + PHY link status OK"

### IP statique du MCU

```
172.20.22.200 / 24
MAC : 02:00:00:00:00:01
```

## Liste des commits — ENC28J60 driver mikroSDK

### ✅ Commit 1 — FAIT

`enc28j60: SPI init + soft reset + EREVID read OK (0x06)`

---

### Commit 2

`enc28j60: MAC config + PHY link status OK`

- Relire MACON1, MACON3, MAADRx et vérifier via UART
- Lire PHSTAT2 → confirmer lien PHY détecté (câble)

### Commit 3

`enc28j60: TX OK — envoi trame Ethernet custom`

- Envoyer une trame EtherType `0x88B5` avec `HELLO_FROM_MCU`
- Vérifier avec Wireshark sur le PC que la trame arrive

### Commit 4

`enc28j60: RX OK — réception trame Ethernet`

- Recevoir une trame et logger EtherType + longueur en UART
- Vérifier avec un `ping` ou trame manuelle depuis le PC

### Commit 5

`enc28j60: ARP reply OK`

- Répondre aux ARP requests du PC
- Vérifier avec Wireshark que le PC résout bien `172.20.22.200`

### Commit 6

`enc28j60: ICMP Echo Reply OK (ping)`

- Répondre aux pings du PC (`ping 172.20.22.200`)
- Logger chaque ping reçu/répondu en UART

### Commit 7

`enc28j60: HTTP server OK — GET / port 80`

- Répondre à une requête HTTP GET avec une page HTML
- Vérifier en ouvrant `http://172.20.22.200` dans un navigateur

### Commit 8 (final)

`enc28j60: driver cleanup — intégration mikroSDK complète`

- Supprimer tous les TODO restants
- Implémenter `get_link_status`, `set_mac`, `get_mac`, `set_ip`, `get_ip`
- Nettoyer le code (commentaires, variables inutilisées, etc.)

---

**Ordre logique de test à chaque étape** : UART log → Wireshark → navigateur. Ne pas passer au commit suivant sans avoir validé le précédent.
Sois concis dans tes reponses.

