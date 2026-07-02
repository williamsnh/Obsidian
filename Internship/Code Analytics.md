UART OK = Correct Initialization with UART 
RESET OK = Correct refresh of the Ethernet chip
SPI OK = Correct SPI Link between MCU and ENC
MICMD=0x00 :  Register that starts the writing/reading of the PHY
MIREGADR=0x00 : this value for PHCON1
CLKRDY OK = Correct stabilization of the clock
EREVID = 0x06 : Correct detection of Ethernet chip

PHY part :
PHCON1=0x000x00 : Configuration of Physical Ethernet behavior (half or full duplex, loopbac, reset PHY, power-down) ; Control register of PHY Ethernet.
PHHID1=0x000x00 : Fix register that should be 0x83
PHSTAT2 = 0x000x00 : PHY Status Register 2 (Link STATUS)
	LINK DOWN - verifier le cable RJ45 = no sign of ethernet cable
	Link = DOWN : same


