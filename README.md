# rainfall

CTF project based on ELF patching and disassembling on i386 systems.

##  Description

This project follows the `snow-crash` CTF challenge where we explored binary exploitation, reverse engineering, and disassembling techniques. In this project, we will build upon that knowledge and introduce new concepts related to ELF patching and advanced disassembly techniques.

Again, the project is structured as a series of levels, each focusing on a specific aspect of ELF binaries and disassembly. The objective on each level is to manage to read the `.pass` file with the with the "levelX" user account of the next level (X = number of the next level).

The ".pass" file is located at the home directory of each (level0 excluded) user.

##  Setup

To set up the environment for this project, you need to start a VM using the provided ISO file. The VM is pre-configured with all the necessary tools and dependencies required for the challenges, all you need to do is confiure the network settings to use a bridged adapter so you can access the VM from your host machine via ssh on port 4242:

```bash
nicolas@pop-os:~$ ssh level0@192.168.1.44 -p4242
	  _____       _       ______    _ _ 
	 |  __ \     (_)     |  ____|  | | |
	 | |__) |__ _ _ _ __ | |__ __ _| | |
	 |  _  /  _` | | '_ \|  __/ _` | | |
	 | | \ \ (_| | | | | | | | (_| | | |
	 |_|  \_\__,_|_|_| |_|_|  \__,_|_|_|

                 Good luck & Have fun

  To start, ssh with level0/level0 on 192.168.1.44:4242
level0@192.168.1.44's password: 
```

The password for the "level0" user is provided in the subject's PDF.

