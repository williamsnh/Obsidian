## Résumé — Portage ENC28J60 mikroSDK : STM32 → PIC24

**Contexte** : librairie SPI Ethernet mikroSDK, carte UNI-DS v8. Fonctionnelle sur **STM32F429ZIT6** (ARP/ICMP/TCP-HTTP OK). Portage en cours vers **PIC24EP512GU814** (XC16), actuellement bloqué.

**Architecture** : `spi_ethernet.c/h` (générique) / `spi_ethernet_enc28j60.c/h` (driver chip) / `main.c` (logique appli, ARP/ICMP/TCP/HTTP).

### Corrections déjà appliquées pour le PIC24 (fichiers actuels fournis)

1. **`data` → `buf`** partout (mot réservé XC16) — fait dans les 4 fichiers.
2. **`log_cfg.is_interrupt = 0;` commenté** dans `main.c` — champ absent de `log_cfg_t` sur cette implémentation PIC24 (`log_cfg_t` n'a qu'un champ `level`). _(Note : la version conditionnelle `#ifdef __XC16__` proposée n'a pas encore été réintégrée dans le fichier actuel — la ligne est juste commentée en dur.)_

### Problème actuel — bloquant

Après flash, plus d'erreur de compilation, mais :

- **Aucun affichage UART**
- **Le programme freeze avant même le premier `log_printf`** (pas dans `log_init`, qui est trivial)

### Diagnostic en cours

- `log_printf` (PIC24) appelle `vfprintf_me(&debugStdOut, f, ap)` — `debugStdOut` vient du runtime XC16 (`<cstdio.h>`), pas de mikroSDK.
- Hypothèse : redirection stdout→UART non configurée (macro `__C30_UART` absente/à vérifier dans les propriétés projet NECTO).
- Mais comme le blocage survient **avant** le premier `log_printf`, l'UART n'est probablement pas la cause racine.

### Pistes non encore vérifiées (prochaines étapes)

1. **Watchdog Timer (FWDT)** potentiellement actif par défaut sur PIC24 → suspect principal pour un freeze/reset en boucle. À désactiver dans les Configuration Bits du projet NECTO.
2. **Désassemblage** (Debug → Disassembly) à l'endroit exact du blocage — pas encore consulté.
3. Compiler en **`-O0`** pour fiabiliser la correspondance ligne↔breakpoint.
4. Isoler en commentant `log_init` et le premier bloc `log_printf` pour voir si le blocage persiste (test demandé, résultat non encore rapporté : l'utilisateur confirme juste "n'atteint jamais le premier log_printf").
5. Vérifier budget pile/RAM PIC24 vs buffers statiques (~2,9 Ko cumulés).

### Autres points à valider plus tard (portage général)

- Vitesse SPI (1 MHz) vs horloge PIC24
- Mapping MikroBUS (`MIKROBUS_2`)
- PPS (Peripheral Pin Select) si config manuelle nécessaire

**Prochaine action recommandée** : vérifier/désactiver le Watchdog dans les Configuration Bits, puis inspecter le désassemblage au point de blocage.