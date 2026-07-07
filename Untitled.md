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

