+++
title = "Рецепт за IPv6 рутер на штапићу – дио први"
date = "2022-02-17"
author = ""
authorTwitter = "" #do not include @
cover = "/posts/recept_ipv6_ruter_1a.jpg"
tags = []
keywords = []
description = "Како да, помоћу *Raspberry Pi*-ја и 6in4 тунела, направите нешто налик [VIIRB](https://ungleich.ch/u/products/viirb-ipv6-box/)-у (*VPN IPv6 IoT Router Box*) и обезбиједите IPv6 у вашој кућној мрежи."
showFullContent = false
Toc=true
readingTime = false
+++

> Како да, помоћу *Raspberry Pi*-ја и 6in4 тунела, направите нешто налик [VIIRB](https://ungleich.ch/u/products/viirb-ipv6-box/)-у (*VPN IPv6 IoT Router Box*) и обезбиједите IPv6 везу у вашој кућној мрежи.

## Потребни састојци

* Интернет веза и јавна[^ipaddr] IPv4 адреса;

[^ipaddr]: Ваша IPv4 не мора бити статичка, али мора бити јавна. Ако је WAN адреса на вашем рутеру из опсега `100.64.0.0`-`100.127.255.255`, то значи да сте иза провајдеровог CGNAT-а и да ово неће радити, јер нећете моћи да направите тунел ка *Tunnelbroker*-у. У том случају остаје вам могућност да узмете неки VPS и да ка њему правите тунел. За више детаља о CGNAT-у погледајте [овај чланак](https://milankragujevic.com/sta-je-cgnat).

* тунел и /64 IPv6 префикс који можете бесплатно добити од [*HE Tunnelbroker*](https://tunnelbroker.net/)-а након регистрације;

* неки Линукс уређај на локалној мрежи који ће да вам служи као IPv6 рутер. То може бити ваш постојећи рутер (ако на њему имате *OpenWRT* или нешто слично), *Raspberry Pi*, рачунар, па и виртуелна машина...

* [`systemd-networkd`](https://www.freedesktop.org/software/systemd/man/systemd.network.html) (и [`systemd-resolved`](https://www.freedesktop.org/software/systemd/man/systemd-resolved.service.html)) за подешавање мреже.

Овдје ћемо користити *Raspberry Pi 3B+* са инсталираним [*Arch Linux ARM*](https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3)-ом. Он ће бити задужен за IPv6, док ће већ постојећи рутер да барата IPv4 саобраћајем.

## Качење *Raspberry*-ја на LAN

Задајмо нашем будућем рутеру статичку LAN адресу у `/etc/systemd/network/eth0.network`:

```systemd
[Match]
Name=eth0

[Network]
Address=192.168.0.2/24
Gateway=192.168.0.1
```

Учитајмо ову конфигурацију -- `systemctl restart systemd-networkd`. Наш уређај би требало да аутоматски добије и IPv6 линк-локалну адресу. Препознаћете је по томе што почиње са `fe80::` и служи, између осталог, за комуникацију и аутоконфигурацију у локалној мрежи:
```shell-session
# ip addr
...
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether f6:70:52:b4:3d:64 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/24 brd 192.168.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f470:52ff:feb4:3d64/64 scope link 
       valid_lft forever preferred_lft forever
...
```
Након пинговања мултикаст адресе `ff02::1` добићемо одговоре од осталих IPv6 уређаја у нашој мрежи:
```shell-session
# ping -I eth0 ff02::1
PING ff02::1(ff02::1) from :: eth0: 56 data bytes
64 bytes from fe80::9854:b4ff:fe01:d160%eth0: icmp_seq=1 ttl=64 time=0.509 ms
64 bytes from fe80::8c04:97ff:fe1e:5b20%eth0: icmp_seq=1 ttl=64 time=2.14 ms
64 bytes from fe80::2047:d1ff:fec2:1802%eth0: icmp_seq=1 ttl=64 time=2.24 ms
^C
--- ff02::1 ping statistics ---
1 packets transmitted, 1 received, +2 duplicates, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.509/1.631/2.244/0.794 ms
```
Сачувајмо ову адресу, јер ће нам ваљати и касније:
```shell-session
# echo 'ff02::1 ip6-neighbors' >> /etc/hosts
```

## Подешавање тунела

Након што смо се регистровали на [tunnelbroker.net](https://tunnelbroker.net/), направили нови тунел и уписали нашу јавну IPv4 адресу као једну од крајњих тачака тунела, биће нам додијељене:

* двије IPv6 адресе за тај тунел (једна -- `2001:db8:aaaa:f::1` -- је адреса сервера, а друга -- `2001:db8:aaaa:f::2` -- је адреса нашег будућег рутера),
* и један `2001:db8:aaab:f::/64` префикс одакле ћемо додјељивати адресе уређајима на нашој мрежи.

{{< image src="/posts/recept_ipv6_ruter_1b.png" alt="Мрежни дијаграм" position="center" >}}

Тунел ћемо подесити у `/etc/systemd/network/sit-he.netdev` и `sit-he.network`:

{{< code language="systemd" title="sit-he.netdev" id="sit-he.netdev" >}}
[NetDev]
Name=sit-he
Kind=sit
# Ако вам веза буде нестабилна, можете испод
# ставити неку мању вриједност. Само не мању
# од 1280.
MTUBytes=1480

[Tunnel]
Local=192.168.0.2
# Адреса tunnelbroker-овог сервера којег сте
# изабрали приликом прављења тунела.
Remote=216.66.87.14
TTL=255
{{< /code >}}

{{< code language="systemd" title="sit-he.network" id="sit-he.network" >}}
[Match]
Name=sit-he

[Network]
Address=2001:db8:aaaa:f::2/64
Gateway=2001:db8:aaaa:f::1
DNSOverTLS=yes
DNS=2001:470:20::2#ordns.he.net
{{< /code >}}

Желимо да се овај тунел подигне након подизања и конфигурације `eth0` интерфејса, што постижемо са:
```shell-session
# echo 'Tunnel=sit-he' >> /etc/systemd/network/eth0.network
```
Рестартујмо `networkd` и тестирајмо наш тунел:
```shell-session
# systemctl restart systemd-networkd.service
# ip addr show dev sit-he
5: sit-he@eth0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
    link/sit 192.168.0.2 peer 216.66.87.14
    inet6 2001:db8:aaaa:f::2/64 scope global 
       valid_lft forever preferred_lft forever
       
# ping -I sit-he ip6-neighbors 
PING ip6-neighbors(ip6-neighbors (ff02::1)) from :: sit-he: 56 data bytes
64 bytes from tunnelnik014.tunnel.tserv1.bud1.ipv6.he.net (2001:db8:aaaa:f::1): icmp_seq=1 ttl=64 time=65.3 ms
64 bytes from tunnelnik014.tunnel.tserv1.bud1.ipv6.he.net (2001:db8:aaaa:f::1): icmp_seq=2 ttl=64 time=67.6 ms
^C
--- ip6-neighbors ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 65.304/66.445/67.586/1.141 ms

# ping google.com
PING google.com(mct01s09-in-x0e.1e100.net (2a00:1450:4018:800::200e)) 56 data bytes
64 bytes from mct01s09-in-x0e.1e100.net (2a00:1450:4018:800::200e): icmp_seq=1 ttl=117 time=188 ms
^C
--- google.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 187.548/187.548/187.548/0.000 ms
```

Пошто смо пинговали неку удаљену IPv6 адресу, знамо да наш тунел ради и да смо успјешно „спровели“ IPv6 до нашег *Raspberry*-ја :) Сад то треба да подјелимо и са остатком мреже.

## Прављење рутера

Да бисмо, коначно, од нашег *RPi*-ја направили рутер, морамо још једном измијенити `eth0.network`:

{{< code language="systemd" title="eth0.network" id="eth0.network" >}}
[Match]
Name=eth0

[Network]
Address=192.168.0.2/24
Gateway=192.168.0.1
Tunnel=sit-he
# eth0 интерфејсу додјељујемо IPv6 адресу:
Address=2001:db8:aaab:f::2/64
# Подешавамо адресу IPv6 DNS сервера на који 
# ћемо упутити остале уређаје у нашем LAN-у:
DNSOverTLS=yes
DNS=2001:470:20::2#ordns.he.net
DNS=74.82.42.42#ordns.he.net
# Кажемо *RPi*-ју да ради искључиво као
# IPv6 рутер, пошто IPv4 рутер већ имамо:
IPForward=ipv6
# И кажемо му да се хвали на сав глас да је 
# рутер (RA = Router Advertisements), како би
# то знали и остали уређаји на мрежи:
IPv6SendRA=yes

[IPv6SendRA]
UplinkInterface=sit-he
RouterLifetimeSec=3600
EmitDNS=yes

[IPv6Prefix]
# Префикс из којег ће уређаји на LAN-у узимати
# адресе:
Prefix=2001:db8:aaab:f::/64
{{< /code >}}

Провјеримо да ли су ова подешавања добра и да ли смо добили нову адресу на нашем `eth0` интерфејсу:
```shell-session
# systemctl restart systemd-networkd.service
# ip addr show dev eth0 
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether f6:70:52:b4:3d:64 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.2/24 brd 192.168.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:db8:aaab:f::2/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::f470:52ff:feb4:3d64/64 scope link 
       valid_lft forever preferred_lft forever
```
И оно што је најбитније за рутер – провјеримо имамо ли добре руте:
```shell-session
# ip -6 route list 
::1 dev lo proto kernel metric 256 pref medium
2001:db8:aaaa:f::/64 dev sit-he proto kernel metric 256 pref medium
2001:db8:aaab:f::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
default via 2001:db8:aaaa:f::1 dev sit-he proto static metric 1024 pref medium
```
Руте нам кажу следеће:

- сваки IPv6 пакет који стигне до *RPi*-ја, а адресиран је за `2001:db8:aaaa:f::/64` (тунел) ће бити прослијеђен кроз `sit-he` интерфејс (којим смо и повезани са тунелом), 

- пакет који је адресиран за нашу `2001:db8:aaab:f::/64` локалну мрежу ће бити послат кроз `eth0` интерфејс,

- остали пакети ће бити прослијеђени *Тunnelbroker*-овој машини на другом крају тунела (тј. на адресу `2001:db8:aaaa:f::1`), па нек она гледа шта ће даље с њима.

Ово је управо то што и желимо да наш нови рутер ради. Остало је још да провјеримо да ли су IPv6 адресе додијељене и осталим уређајима у локалној мрежи:
```shell-session
[Nik014@laptop ~]$ ip addr show dev wlan0 
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
...
    inet6 2001:db8:aaab:f:9854:b4ff:fe01:d160/64 scope global dynamic mngtmpaddr noprefixroute 
       valid_lft 3146sec preferred_lft 1346sec
...
```
Сад можемо отићи на https://ipv6-test.com/ и „уживати“ у новој IPv6 вези :)

## *ddclient* за динамичке IPv4 адресе

Ако немате статичку IPv4 адресу, морате још инсталирати и подесити `ddclient` да повремено освјежава ту адресу на вашем крају `sit-he` тунела:
```shell-session
# pacman -S ddclient
...
# cat << _EOF_ >> /etc/ddclient/ddclient.conf

##
## HE Tunnelbroker (https://www.tunnelbroker.net)
##
protocol=dyndns2
use=web
web=checkip.dns.he.net
server=ipv4.tunnelbroker.net
ssl=yes
login=Vaš_username_na_tunnelbrokeru
password=Update_key_token_za_vaš_tunel
Tunnel_ID_broj
_EOF_
# systemctl enable --now ddclient.service
```
Број и *update key* за ваш тунел ћете наћи на *Tunnelbroker* сајту.

## Шта даље?

Осим основне (рутирање саобраћаја), вјероватно ћемо хтјети да наш *RPi* рутер обавља и неке додатне функције. У наредним постовима ћемо видјети како да на њему подесимо DHCPv6 сервер[^dhcp], firewall и DNS сервер за локалну мрежу.

[^dhcp]: [A. Yourtchenko: *A comparison between the DHCPv6 and RA based host configuration*](https://tools.ietf.org/id/draft-yourtchenko-ra-dhcpv6-comparison-00.html)

За сада можете да „пентестирате“ вашег провајдера, тако што ћете *RPi* повезати директно са кабловским/DSL модемом (или тако што ћете рутер ставити у *bridge* мод) а у горњим подешавањима ставити вашу јавну IPv4 адресу умјесто `192.168.0.2`. Овако ће *RPi*, умјесто уређајима у кућној мрежи, слати *Router Advertisements* сусједним рутерима на провајдеровој страни мреже. Ако они прихвате RA које шаље ваш *RPi* и од њега узму IPv6 адресу, значи да су подложни [*rogue router*](https://gcn.com/cybersecurity/2013/08/easy-to-use-attack-exploits-ipv6-traffic-on-ipv4-networks/281426/) нападу.


## Лектира


1. [`systemd.network` man page – freedesktop.org](https://www.freedesktop.org/software/systemd/man/systemd.network.html)

3. [`systemd.netdev` man page – freedesktop.org](https://www.freedesktop.org/software/systemd/man/systemd.netdev.html)

4. [How-to: IPv6 Neighbor Discovery | APNIC Blog](https://blog.apnic.net/2019/10/18/how-to-ipv6-neighbor-discovery/)
