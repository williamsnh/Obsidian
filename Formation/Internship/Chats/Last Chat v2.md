# Résumé — Stage portage librairie SPI Ethernet (mikroSDK)

## Contexte / objectif

Stage : porter une librairie Ethernet SPI (trames brutes, sans OS réseau) sur mikroSDK, testée sur plusieurs MCU via socket Sibrain interchangeable (carte UNI-DS v8).

Architecture générique :

- `spi_ethernet.h/.c` — couche commune, struct `spi_ethernet_driver_t`
    
- `spi_ethernet_<chip>.c/.h` — un driver par puce
    
- `main.c` unique multi-puces via macro `SPI_ETH_CHIP`
    

Le but est de pouvoir utiliser la même architecture logicielle sur différentes familles de MCU et différents contrôleurs Ethernet.

## Matériel / toolchain

- MCU principal de test : **STM32F429ZIT6** sur carte **UNI-DS v8**
    
- Compilateur : **mikroC AI for ARM**
    
- Autres MCU validés : PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L, ATMEGA6450V8U, PIC18F97J94
    
- Piège compilateur récurrent : ne jamais nommer une variable `data` → utiliser `val` / `value`
    

## Avancement des puces

1. ✅ **ENC28J60** — driver complet et validé  
    ARP / ICMP / TCP / HTTP fonctionnels.
    
2. ✅ **W5500** — driver complet et validé  
    ETH Click, socket 0 en MACRAW. Ping / HTTP / ICMP OK.  
    `w5500_get_link_status()` vérifie `VERSIONR == 0x04` avant de considérer `PHYCFGR`, afin d'éviter les faux "LINK UP".
    
3. ✅ **LAN9250** — driver complet et fonctionnel  
    FIFO TX/RX, SPI READ/WRITE, registres 32 bits LSB-first, accès indirect MAC_CSR/MII, etc.
    
4. 🔄 **LAN9252** — en cours  
    ASIC EtherCAT Slave Controller, donc différent des contrôleurs Ethernet classiques.  
    Nécessite un maître EtherCAT comme SOEM + ESI XML.  
    Bring-up SPI avec `BYTE_TEST`, `HW_CFG`, `ID_REV` en cours/finalisation.
    
5. ✅ **W6100** — **driver maintenant fonctionnel et validé**  
    Le W6100 fonctionne correctement avec l'architecture `spi_ethernet_driver_t`.  
    Cette partie peut donc être considérée comme terminée.
    

## Cas particulier : WIZ-IP20

Le **WIZ-IP20** est un module UART → Ethernet autonome et ne doit pas être forcé dans l'abstraction SPI.

Architecture retenue :

- `spi_ethernet_wizip20.c/.h`
    
- communication UART
    
- commandes AT
    
- séparation dans `main.c` avec `#if SPI_ETH_CHIP == WIZ_IP20`
    

Développement prévu de manière incrémentale :

**UART → reset → AT/OK → version → IP → TCP → data**

## Prochaine étape : DHCP

Maintenant que le **W6100 fonctionne correctement**, l'objectif suivant est de travailler sur le **DHCP client**.

L'idée est de remplacer l'IP statique actuelle par une configuration obtenue automatiquement auprès d'un serveur DHCP.

Le principe sera :

```text
DHCPDISCOVER
      ↓
DHCPOFFER
      ↓
DHCPREQUEST
      ↓
DHCPACK
      ↓
IP / subnet mask / gateway / DNS obtenus
```

Le point de départ est donc de comprendre précisément **où intégrer DHCP dans l'architecture actuelle** et comment utiliser les fonctions UDP déjà présentes.

### État actuel

- Ethernet / SPI : ✅
    
- Drivers ENC28J60 / W5500 / LAN9250 / W6100 : ✅
    
- UDP : déjà commencé/présent dans le projet
    
- DHCP : ⏳ **prochaine tâche**
    
- L'utilisateur ne sait pas encore exactement quelles étapes implémenter ni dans quel ordre.
    

**La prochaine chose à faire est donc de prendre le code DHCP existant du projet et de construire l'implémentation étape par étape**, plutôt que de coder directement les quatre messages DHCP au hasard.