
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
![](images/2026-03-09%2011_26_13-Windows%2011%20-%20VMware%20Workstation.png)
