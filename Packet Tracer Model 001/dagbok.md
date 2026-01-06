# 2026-01-06
En vacker dag ska jag bli klar med detta. Uppgiften är sedan länge inlämnad i skolan men jag återskapar den här från minne. Och med pauserna tar det lite tid.

Nu är det dags att implementera möjlighet att använda SSH för att koppla upp sig mot router, så jag använder följande kommando:

    en
    conf t
    ip domain-name GBG
    crypto key generate rsa general-keys modulus 4096
    ip ssh version 2
    line vty 0 15
    exec-timeout 2
    login local
    transport input ssh
    exit
    enable secret 1234
    username admin enable secret 1234


Dags att applicera lite ACLs så att bara administratörer kommer åt routern:

    en
    conf t
    ip access-list standard ADMIN
    permit 172.16.2.128 0.0.0.63
    remark allow admin and deny everyone else
    exit
    line vty 0 15
    access-class ADMIN in

Nu ska jag göra en ACL så att gästnätverket är isolerat från de övriga datorerna på nätverket och framtida nätverk. Men de ska kunna komma åt internet. Följande kod kör jag på routerns CLI:
    
    en
    conf t
    ip access-list extended GUEST
    deny ip 172.16.2.64 0.0.0.63 172.16.2.0 0.0.0.63
    deny ip 172.16.2.64 0.0.0.63 172.16.2.128 0.0.0.63
    deny ip 172.16.2.64 0.0.0.63 192.168.0.0 0.0.255.255
    deny ip 172.16.2.64 0.0.0.63 10.0.0.0 0.255.255.255
    permit ip any any
    remark guests can only access internet and eachother
    exit
    interface gig0/0.20
    ip access-group GUEST in






# 2025-11-21
Tog en stund att återkomma hit.
Men nu implementerar jag SVI och en enkel ACL på alla switchar i nätverket.

Exempelkod för att uppnå en fungerande SVI som bara går att kontakta via SSH version 2:
    
    en
    conf t
    int vlan 99
    ip address 172.16.2.131 255.255.255.192
    description SVI
    exit
    ip default-gateway 172.16.2.129
    ip domain-name GBG
    crypto key generate rsa general-keys modulus 2048
    ip ssh version 2
    line vty 0 15
    exec-timeout 2
    login local
    transport input ssh
    exit
    enable secret 1234
    username admin secret 1234


Dags att applicera lite ACLs så att bara administratörer kommer åt SVI:

    en
    conf t
    ip access-list standard ADMIN
    permit 172.16.2.128 0.0.0.63
    remark allow admin and deny everyone else
    exit
    line vty 0 15
    access-class ADMIN in

    
![SSH Success!](<Screenshots/Skärmbild 2025-11-21 ssh success.png>)


# 2025-11-06 uppdatering

Problem med att få upp det här på github, försöker igen.

Och igen.

# 2025-11-06
![Orörd packet tracer-fil från skolan](<Screenshots/Skärmbild 2025-11-06 Orörd.png>)

Jag börjar med att konfigurera switchar på GBG-sidan, se kod längre ner.
Vi skapar alltså alla vlan, 10 för kontor, 20 för gäst, 99 för admin samt 999 som ska användas senare för övrig trafik.

Konfigurerar trunkportarna som ska gå mellan switchar och routrar så att alla vlan kan kommuniceras ordentligt.
Efter det här så installerar jag end points och ställer in dhcp på routerns olika subinterfaces. Nu öppnar jag portar på switcharna till end points och konfigurerar dem allt efter som för att hålla ordning på de olika vlanen.

Jag gör samma vända på STH-sidan så vi har en spegelbild, mer eller mindre.

Nu börjar det vara dags för att skapa och konfigurera den virtuella tunneln vi ska ha. Efter det är klart så gör vi två statiska IP routes.

Nu är allt på plats, det  går bra att pinga mellan de olika siterna. 

Kod för olika ändamål till switchar och router:

    vlan 10
    name Office
    vlan 20
    name Guest
    vlan 99
    name Admin
    vlan 999
    name Blackhole
    interface range fa0/1-24
    switchport access vlan 999
    description Not in use
    shutdown
    interface range gig0/1-2
    switchport access vlan 999
    description Not in use
    shutdown
    interface gig0/1
    switchport mode trunk
    switchport trunk native vlan 100
    switchport trunk allowed vlan 10-100
    no shutdown

    ip dhcp pool vlan10
    default-router 172.16.2.1
    network 172.16.2.0 255.255.255.192

    ip dhcp excluded-address 172.16.2.129 172.16.2.133

    interface tunnel 1
    ip address 192.168.255.2 255.255.255.252
    tunnel destination 80.80.80.2
    tunnel mode gre ip
    tunnel source Serial 0/3/0

    ip route 10.0.1.0 255.255.255.0 192.168.255.1
    ip route 0.0.0.0 0.0.0.0 Serial 0/3/0

Efter dagens arbete ser det ut ungefär såhär
![Efter dagens arbete](<Screenshots/Skärmbild 2025-11-06 Efter dagens arbete.png>)<br><br>
Och vi verkar kunna pinga mellan GBG och STH!<br>
![alt text](<Screenshots/Skärmbild 2025-11-06 ping mellan sites.png>)