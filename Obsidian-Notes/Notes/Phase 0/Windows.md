
### Configuration Windows 
#### Modification de l'hyperviseur (VMware) :
- Trouve le fichier de configuration de la VM : `.vmx`
	- `ethernet0.virtualDev = "e1000e"` -> `ethernet0.virtualDev = "vmxnet3"`
#### Le Tagging
- **Gestionnaire de périphériques > Cartes réseau**.
	- **Propriétés** > Onglet **Avance**
		- **VLAN ID** = 20
#### Validation 
- cmd : `ipconfig`

``` shell
PS C:\Users\Windows> ipconfig
Windows IP Configuration
Ethernet adapter Ethernet1 2:
   Connection-specific DNS Suffix  . : charif-labs.tech
   Link-local IPv6 Address . . . . . : fe80::c245:c27d:d02a:5c70%9
   IPv4 Address. . . . . . . . . . . : 10.0.20.100
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.0.20.1
```