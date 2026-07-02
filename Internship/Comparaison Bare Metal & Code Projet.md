**Bare metal** :
![[Pasted image 20260702083324.png|349]]
sur enc_phy_read, renvoie vers enc_write_reg 

![[Pasted image 20260702090910.png|357]]
Lorsque mb1_print_hex(high); mb1_print_hex(low); 


![[Pasted image 20260702122535.png|407]]
L213 : *high = enc_read_reg(MIRDH & 0x1F);


**Code Project :**
![[Pasted image 20260702123405.png]]
L509 : *enc28j60_write_reg(MICMD, 0x00);


![[Pasted image 20260702123257.png]]
L512 : *high = enc28j60_read_reg(MIRDH);

![[Pasted image 20260702130027.png]]

