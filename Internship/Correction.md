- ==change mb1 to logger function==
- ==initialize spi pins throught mikrobus==
==-#define IPSDISPLAY2_MAP_MIKROBUS( cfg, mikrobus ) \==

==cfg.miso = MIKROBUS( mikrobus, MIKROBUS_MISO ); \==

==cfg.mosi = MIKROBUS( mikrobus, MIKROBUS_MOSI ); \==

==cfg.sck = MIKROBUS( mikrobus, MIKROBUS_SCK ); \==

==cfg.cs = MIKROBUS( mikrobus, MIKROBUS_CS ); \==

==cfg.rst = MIKROBUS( mikrobus, MIKROBUS_RST ); \==

==cfg.bck = MIKROBUS( mikrobus, MIKROBUS_AN ); \==

==cfg.dc = MIKROBUS( mikrobus, MIKROBUS_INT )==

- ==remove spi_master_set_chip_select_polarity(SPI_MASTER_CHIP_SELECT_DEFAULT_POLARITY);==
- ==ne pas oublier que je vais tester avec plusieurs MCU donc main.c doit appele a chaque fois spi_ethernet_... et pas enc28j60_...==
- ==respecter la convention mikrosdk en laissant des espaces entre les parentheses sur :==
==---> main.c==
==--> spi_ethernet.c==
==--> spi_ethernet.h==
==--> spi_ethernet_enc28j60.c==
==--> spi_ethernet_enc28j60.h==

- Tester le code sur plusieurs MCU (toujours sur UNI DS v8) :
==--> Origin = STM32F429ZIT6==
==--> PIC24EP512GU814==
==--> dsPIC33FJ256GP710A==
--> PIC18F97J94
--> PIC32MZ2048EFH144
--> PIC32MX795F512F