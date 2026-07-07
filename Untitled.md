- change mb1 to logger function
- initialize spi pins throught mikrobus
-#define IPSDISPLAY2_MAP_MIKROBUS( cfg, mikrobus ) \

cfg.miso = MIKROBUS( mikrobus, MIKROBUS_MISO ); \

cfg.mosi = MIKROBUS( mikrobus, MIKROBUS_MOSI ); \

cfg.sck = MIKROBUS( mikrobus, MIKROBUS_SCK ); \

cfg.cs = MIKROBUS( mikrobus, MIKROBUS_CS ); \

cfg.rst = MIKROBUS( mikrobus, MIKROBUS_RST ); \

cfg.bck = MIKROBUS( mikrobus, MIKROBUS_AN ); \

cfg.dc = MIKROBUS( mikrobus, MIKROBUS_INT )

- remove spi_master_set_chip_select_polarity(SPI_MASTER_CHIP_SELECT_DEFAULT_POLARITY);
- ne pas oublier dque je vais tester avec plusieurs MCU donc main.c doit appele a chaque fois spi_ethernet_... et pas enc28j60_...