- **Adresses :** `10.0.10.2/29`
- **Passerelle (Routes/Gateway) :** `10.0.10.1`

#### Fichier 
/etc/netplan/50-cloud-init.yaml
#### L'interface
ens33 / `ip a`
#### Netplan YAML only Trunk
``` yaml
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 10.0.10.2/29
      routes:
        - to: default
          via: 10.0.10.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 10.0.10.1
```

#### Netplan YAML trunk and WAN
``` YAML
network:
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 10.0.10.2/29
      routes:
        - to: default
          via: 10.0.10.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 10.0.10.1
    ens37:
      dhcp4: true
```

#### Application de la configuration
`sudo netplan apply`