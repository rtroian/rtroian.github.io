---
title: "Archer C5 FailSafe"
date: 2019-05-01
tags:
  - FailSafe
  - OpenWrt
  - TP-link
  - Archer
header:
  # image: "/images/8bitheart.jpg"
# excerpt: "FLOSS, OpenWRT, Mesh, InfoSec"
---

No caso de termos um Archer C5 sem responder a nenhuma das portas cabeadas e sem fio, os procedimentos para tentar seu resgate são os seguintes.
Neste caso em especifico estamos usando o modelo v1.2, que de acordo com o site do OpenWRT tem as seguintes caracteristicas:


>Dual Band (concurrent) and Gigabit Ethernet. Advertised as 1750 Mbps. It has simultaneous Triple-Stream (3×3) radios on both 2.4GHz and 5 GHz Bands. It supports 802.11n in 2.4GHz for 450Mbps throughput and IEEE 802.11ac (draft) for 1300Mbps throughput in 5GHz.
>The Archer C5 v1.x: available worldwide, has 3 internal (2.4GHz) and 3 external (5GHz) antennas.
>The Archer C5 v2.0: available worldwide, has ?? internal (2.4GHz) and 2 external (5GHz) antennas.
> <cite><a href="https://openwrt.org/toh/tp-link/archer-c5-c7-wdr7500">OpenWRT Site</a></cite>


Para entrar em Failsafe Mode no Archer C5 ou no Archer C7, é necessário pressionar o botão WPS/Reset rapidamente após iniciar o roteador, mas ao contrário do normal que se mantem o botão pressionado, nesse caso apertar alternadamente é o que funciona!

Apenas **Parar** quando o segundo LED (um asterisco) começar a **piscar** rapidamente.

Depois em tese é só seguir o processo normal como visto [aqui](https://openwrt.org/docs/guide-user/troubleshooting/failsafe_and_factory_reset).

Mas é aqui que as coisas ficam mais chatas, as vezes o roteador nao esta com nada ativado e o link não se mantém, ai precisamos conectar via cabo na porta lan, limpar todas as rotas e endereços, conferir se a placa esta ativa, adicionar o endereço e ai sim começar o upgrade, então vamos ao passo a passo:

1. desativar gerenciador de rede:
```
# systemctl stop NetworkManager
```
1. desconectar todos cabos e wifis e limpar todas rotas de conexões:
```
  > # ip r
  > # ip r del default .... (copiar da saida acima)
```
1. conferir se nao temos um ip na mesma linha do que precismos:
```
> # ip a
> # ip a del ... dev SEUDEV
```
1. conferir se o link nao entrou em estado desligado:
```
> # ip l
> # ip l set dev SEUDEV up
```
1. adicionar o endereço certo para comunicar, meu caso usando o 192.168.1.100:
```
> # ip a add 192.168.1.100/24 dev SEUDEV
```
1. e se tudo deu certo, o comando ping para 192.168.1.1 deve estar funcionando, em caso cotnrário, conferir todos os passos :)
```
> ping 192.168.1.1
PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.731 ms
64 bytes from 192.168.1.1: icmp_seq=2 ttl=64 time=0.762 ms
```
1. Agora podemos ir ao recovery propriamente dito:
```
ssh root@192.168.1.1 -v
OpenSSH_8.0p1, OpenSSL 1.1.1c  28 May 2019
debug1: Reading configuration data /etc/ssh/ssh_config
debug1: Connecting to 192.168.1.1 [192.168.1.1] port 22.
debug1: Connection established.
```
1. Responder sim ao fingerprint e seguir a conexão:
```
-----------------------------------------------------
 OpenWrt SNAPSHOT, r9662-3411399
 -----------------------------------------------------
================= FAILSAFE MODE active ================
special commands:
* firstboot	     reset settings to factory defaults
* mount_root	 mount root-partition with config files

after mount_root:
* passwd			 change root's password
* /etc/config		    directory with config files

for more help see:
https://openwrt.org/docs/guide-user/troubleshooting/
- failsafe_and_factory_reset
- root_password_reset
=======================================================
root@(none):~#

```
Se deu certo até agora, entramos em uma tela assim (no caso de ja ter existido uma instalação anterior, claro...)

Agora vamos reverter para a configuração de fabrica:

```
root@(none):~# firstboot
This will erase all settings and remove any installed packages. Are you sure? [N/y]
y
/dev/mtdblock4 is not mounted
/dev/mtdblock4 will be erased on next mount
root@(none):~# reboot & exit
```

E voltamos a ter uma imagem funcional, no meu caso uma LiMe mas poderia ser qualquer outro sabor de openwrt \0/

E agora temos uma versão funcionando do openwrt... vamos a parte de voltar a firmware original em outro texto!

Happy FailSafing!


pagina do archer c5 e c7 no [site do OpenWrt](https://openwrt.org/toh/tp-link/archer-c5-c7-wdr7500)
