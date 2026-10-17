## Résumé de la conversation — Préparation présentation orale stage Mikroe

**Contexte général**

- Stage 12 semaines chez MIKROELECTRONIKA, poste Embedded Software Developer
- Projet software terminé en 8 semaines (au lieu de 12) → demande volontaire de passer en Embedded Hardware pour les 3 dernières semaines (PCB design, choix composants, Altium)
- Présentation orale à faire devant des gens de Mikroe (grand public, pas trop technique)

**Plan final du sommaire (6 sections numérotées 01-06 sur la slide Table of Contents)**

1. **About Me** — Who I am, my background & studies
2. **My Internship** — Software Development / PCB Design & Hardware (slide "Internship Objectives" avec 3 blocs : Develop software skills / Explore hardware design / Grow professionally — sert d'intro globale, pas touchée)
3. **Software Project** — Architecture, implementation & results
4. **Hardware Project** — Component selection & PCB design
5. **What I learned** — Technical & professional growth
6. **My experience in Serbia** — Life, culture & personal experience

**Slide "What I learned" — contenu validé (version courte, l'essentiel à l'oral)**

_Technical Skills:_

- SPI / I2C communication buses
- Multi-MCU testing & adaptation (UNI DS v8)
- Debugging & integration of new Click boards
- PCB design & Altium basics

_Professional Skills:_

- Autonomy & ownership of a full project
- Proactive communication to fully understand project context
- Iterating on code through review feedback
- (à considérer : Proactivity — requested to extend scope into hardware)

_Personal takeaway:_ à finaliser — proposition : "This internship confirmed my passion for embedded systems — and sparked new curiosity for hardware design." (dépend si le hardware donne envie d'en refaire ou juste une découverte ponctuelle — question encore ouverte)

**Section Software Project — détails techniques (pour rédaction des slides)**

- Projet : librairie SPI Ethernet générique pour mikroSDK, portable sur plusieurs MCU/contrôleurs Ethernet sans changer l'archi logicielle
- Architecture : `spi_ethernet.h/.c` (couche commune, struct `spi_ethernet_driver_t` avec callbacks) + un driver par puce (`spi_ethernet_<chip>.c/.h`) + `main.c` unique multi-puces via macro
- Process suivi : code bare-metal exploratoire → librairie générique → premier driver (ENC28J60) → validation sur STM32F429ZIT6 (carte UNI-DS v8) → portage sur autres MCU (PIC24, dsPIC33, PIC32, ATmega, PIC18) avec debug d'erreurs de typage/compilateur → ajout de nouveaux chips Ethernet (W5500, LAN9250, LAN9252, W6100) → refactoring/optimisation + doc Doxygen
- Chips Ethernet intégrés : ENC28J60 ✅, W5500 ✅, LAN9250 ✅, LAN9252 (EtherCAT, cas particulier ASIC esclave) 🔄, W6100 🔄
- Cas particulier : ETH WIZ 4 Click / WIZ-IP20 — pas un chip SPI (communique en UART, pile TCP/IP intégrée), donc volontairement exclu de l'abstraction `spi_ethernet_driver_t`, driver séparé
- Fonctions implémentées : init SPI, ARP, ICMP (ping), TCP (mini serveur HTTP raw avec handshake, checksums), dispatch IP
- MCU testés : STM32F429ZIT6, PIC24EP512GU814, dsPIC33FJ256GP710A, PIC32MX795F512L, ATMEGA6450V8U, PIC18F97J94 — tous sur carte UNI-DS v8 (socket Sibrain interchangeable)
- Compilateurs : mikroC AI for ARM, XC8, XC16, GCC, NECTO
- Piège récurrent noté : ne jamais nommer une variable `data` en mikroC (conflit) → toujours `val`/`value`

**Project Overview rédigé (validé) :**

> _"The goal of this project was to develop a generic SPI Ethernet library for mikroSDK — one that works across multiple Ethernet controllers and microcontroller families without changing the application code. Starting from a bare-metal proof of concept on a single chip, I built a layered architecture separating the common API from chip-specific drivers, then extended it to support new Ethernet controllers and MCU families one by one."_

Version courte alternative :

> _"I developed a generic SPI Ethernet library for mikroSDK, designed to work seamlessly across different Ethernet controllers and microcontroller families — without rewriting application code for each combination."_

**Prochaines étapes identifiées (pas encore faites)**

- Rédiger slide "Architecture" (visuel schéma API commune + drivers par puce)
- Rédiger contenu "Testing" section software (l'utilisateur a dit savoir quoi y mettre — liste MCU + Click boards testés)
- Rédiger slide "Results" software project
- Détailler les 4 slides du **Hardware Project** (Why & What, Component Selection, Schematic & Routing Altium, Results) — pas encore rédigées en détail
- Trancher la phrase finale du "Personal takeaway" selon l'intérêt réel pour le hardware