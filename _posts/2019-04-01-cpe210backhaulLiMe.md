---
title: "TP-Link CPE 210 como Backhaul usando LiMe"
date: 2019-04-01
tags: [generic, mesh, openwrt, LiMe, ]
header:
  # image: "/images/8bitheart.jpg"
# excerpt: "FLOSS, OpenWRT, Mesh, InfoSec"
---


A instalação padrão do LibreMesh que compilamos para a CPE 210 v3 vem com a __unica porta de rede__ ativa como __Lan__, assim como possivelmente em outras versões de roteadores com uma unica porta LAN.
Para poder compartilhar internet precisamos ter uma porta WAN ativa, e o bom é que conseguimos  resolver isso mudando algumas configurações no roteador que servirá de borda para a instalação.

---

#### Cenário:

inicialmente o cenário que estamos trabalhando é o seguinte:

```
internet -> router caseiro 192.168.0.1 -> cabo rede  roteador lime 4300 porta wan - wifi lan 4300 -> via mesh -> cpe 210
```

partindo desse cenario, supomos que a CPE esta __acessivel via ssh__ e tem __acesso a internet__

---
#### Ajustes iniciais
Como a compilação que estamos usando ainda não incluiu o pacote de admin, após acessar o roteador CPE 210 por ssh partindo de sua maquina conectada a ele por meio da mesh, iremos atualizar a fonte de pacotes e instalar o gerenciador grafico para auxiliar no processo de visualização:

	ssh root@thisnode.info -v

```bash
opkg update

opkg install luci-mod-admin-full
```

apos o pacote instalado ja sera possivel observar quais interfaces estao disponiveis e quais não atraves do navegador web, acessando:

	http://thisnode.info

e indo na pagina de administração e interefaces:

	Luci - administration - interfaces

no nosso caso nao existia nenhuma interface relacionando a __wan__ e a __eth0__ como interface solo tambem, como observado no 4300 que esta servindo de borda nesse momento.

para isso podemos criar as varias interfaces na mão ou acessar o roteador e criar as interfaces que estao faltando atraves do vi:

	ssh root@thisnode.info

```
vi /etc/config/network
```

basicamente serão adicionadas as interfaces:
```
 	wan, wan6, lm_net_eth0, lm_net_eth0_batadv_dev, lm_net_eth0_batadv_if
```

e para referencia, segue meu arquivo de interfaces que está funcionando:

```
config interface 'loopback'
	option ifname 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config globals 'globals'

config interface 'lan'
	option type 'bridge'
	option proto 'static'
	option ip6assign '60'
	option ipaddr '10.13.155.136'
	option netmask '255.255.0.0'
	option mtu '1500'
	option ifname 'bat0'

config device 'lm_net_br_lan_anygw_dev'
	option type 'macvlan'
	option name 'anygw'
	option ifname 'br-lan'
	option macaddr 'aa:aa:aa:0d:fe:aa'

config interface 'lm_net_br_lan_anygw_if'
	option ifname 'anygw'
	option auto '1'
	option netmask '255.255.0.0'
	option proto 'static'
	option ipaddr '10.13.0.1'
	option ip6addr '2a00:1508:a0d:fe00::1/64'

config rule6 'lm_net_anygw_rule6'
	option src '2a00:1508:a0d:fe00::1/128'
	option lookup '170'

config route6 'lm_net_anygw_route6'
	option interface 'lm_net_br_lan_anygw_if'
	option target '2a00:1508:a0d:fe00::/64'
	option table '170'

config rule 'lm_net_anygw_rule4'
	option src '10.13.0.1/32'
	option lookup '170'

config route 'lm_net_anygw_route4'
	option interface 'lm_net_br_lan_anygw_if'
	option target '10.13.0.0'
	option netmask '255.255.0.0'
	option table '170'

config interface 'lm_net_batadv_dummy_if'
	option ifname 'dummy0'
	option macaddr 'aa:4e:26:df:9b:88'
	option proto 'batadv'
	option mesh 'bat0'

config interface 'lm_net_wlan0_mesh'
	option proto 'none'
	option mtu '1536'
	option auto '1'

config device 'lm_net_wlan0_mesh_batadv_dev'
	option type '8021ad'
	option name 'wlan0-mesh_29'
	option ifname '@lm_net_wlan0_mesh'
	option vid '29'
	option mtu '1532'

config interface 'lm_net_wlan0_mesh_batadv_if'
	option auto '1'
	option ifname 'wlan0-mesh_29'
	option proto 'batadv'
	option mesh 'bat0'

config device 'lm_net_wlan0_mesh_bmx6_dev'
	option type '8021ad'
	option name 'wlan0-mesh_13'
	option ifname '@lm_net_wlan0_mesh'
	option vid '13'
	option mtu '1500'

config device 'lm_net_eth0_batadv_dev'
	option type '8021ad'
	option name 'eth0_29'
	option ifname 'eth0'
	option vid '29'
	option mtu '1532'


config interface 'lm_net_wlan0_mesh_bmx6_if'
	option auto '1'
	option ifname 'wlan0-mesh_13'
	option proto 'static'
	option ipaddr '169.254.155.136'
	option netmask '255.255.255.255'

config interface 'lm_net_eth0'
	option mtu '1500'
	option proto 'none'
	option ifname 'eth0'
	option auto '1'

config interface 'lm_net_eth0_batadv_if'
	option auto '1'
	option proto 'batadv'
	option ifname 'eth0_29'
	option mesh 'bat0'

config interface 'lm_net_eth0'
	option proto 'none'
	option auto '1'
	option ifname 'eth0'
	option mtu '1500'

config interface 'wan'
	option proto 'dhcp'
	option ifname 'eth0'

config interface 'wan6'
	option proto 'none'
	option ifname 'eth0'

```


e adicionar as zonas certas ao firewall, que no router se encontra em /etc/config/firewall e  adicionar ao fim da zona __lan__ a interface lm-net-eth0-batadv-if:


```
config zone
	option name 'lan'
	option input 'ACCEPT'
	option output 'ACCEPT'
	option forward 'ACCEPT'
	option network 'lan lm_net_br_lan_anygw_if lm_net_batadv_dummy_if lm_net_wlan0_mesh_batadv_if lm_net_wlan0_mesh_bmx6_if lm_net_eth0_batadv_if'

```

---
### Testes:

Feitos em varios passos:

#### partindo do router cpe:

desconectar o cabo de internet que estava no outro roteador da malha em wan e conectar ao cpe em wan , __reiniciar__  o  CPE e __desligar__  o outro router.

	ssh root@thisnode.info

- ping partindo do router para o roteador caseiro 192.168.1.1
- ping partindo do router para fora, ex 8.8.8.8
- ping via dns, ex ping riseup.net
- opkg update

- ligar outro router
- ip n para ver vizinhos
- esperar router aparecer (ou monitorar por bmx)
- pingar outro router da malha

#### partindo de computador cliente conectado na cpe

- ping 10.13.0.1
- ping 8.8.8.8
- ping eff.org
- http

se tudo estiver ok até agora significa que o roteador esta com a porta wan ok, agora precisamos conferir os outros nodos da rede:

#### partindo de outro roteador na malha conectado ao cpe

- ligar o outro nodo da rede que antes tinha a wan
- trocar de rede
- conectar pc ao outro roteador
- acessar thisnode.info
- ping 10.13.0.1
- ping ip cpe
- ping 8.8.4.4
- ping qualquernome.com
- http

se tudo isso funcionar, siginifica que __deu certo__!


happy backhauling!

ps: _nos meus testes, com as placas certas mas o firewall não, ele funcionava partindo da cpe mas nao dos outros nodos, ao ajustar o firewall e reiniciar a malha, tudo voltou a vida!_
