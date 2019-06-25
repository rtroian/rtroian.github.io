---
title: "Archer C5 OpenWRT e firmware de fabrica"
date: 2019-05-10
tags:
  - FailSafe
  - OpenWrt
  - TP-link
  - Archer
header:
  # image: "/images/8bitheart.jpg"
# excerpt: "FLOSS, OpenWRT, Mesh, InfoSec"
---
Partindo de um roteador com LibreMesh com problemas no ambiente gráfico, começamos com a questão de devolver a firmware original diretamente ou tentar fazer o caminho de instalação via openwrt?


Nesse caso, como o ambiente gráfico não está funcionando vamos mudar o sistema LibreMesh para um Openwrt "puro" e depois para o original, alterando a firmware (corte do setor de boot)


O link para a  firmware pode ser acessado direto [daqui](https://static.tp-link.com/res/down/soft/Archer_C5_V1.20_150428.zip), e para a pagina dela que mostra que a ultima versao é de 2015 [aqui](https://www.tp-link.com/en/support/download/archer-c5/v1.20/#Firmware).

**Atenção:** Siga esses passos por usa conta e risco!
{: .notice--warning}


1. Começamos acessando o router por shell:
```
> # ssh root@10.13.0.1 -v
```
1. Depois partindo da pasta onde foi salvo o openwrt para o modelo que baixamos:
```
> # scp openwrt-18.06.2-ar71xx-generic-archer-c5-v1-squashfs-sysupgrade.bin root@10.13.0.1:/tmp
openwrt-18.06.2-ar71xx-generic-archer-c
100% 3968KB 734.9KB/s   00:05
```
1. Agora voltando ao router:
```
> # cd /tmp/
```
1. Executamos o procedimento de sysupgrade:
```
root@LiMe-56b747:/tmp# sysupgrade openwrt-18.06.2-ar71xx-generic-archer-c5
-v1-squashfs-sysupgrade.bin
Image metadata not found
Use sysupgrade -F to override this check when downgrading or flashing to vendor firmware
Image check 'fwtool_check_image' failed.
root@LiMe-56b747:/tmp#
root@LiMe-56b747:/tmp# sysupgrade -F openwrt-18.06.2-ar71xx-generic-archer
-c5-v1-squashfs-sysupgrade.bin
Image metadata not found
Image check 'fwtool_check_image' failed but --force given - will update anyway!
Saving config files...
Commencing upgrade. Closing all shell sessions.
debug1: channel 0: free: client-session, nchannels 1
Connection to 10.13.183.71 closed by remote host.
Connection to 10.13.183.71 closed.
Transferred: sent 5488, received 8560 bytes, in 339.8 seconds
Bytes per second: sent 16.1, received 25.2
debug1: Exit status -1
```
1. se tudo deu certo, agora devemos ter um servidor dhcp iniciando no roteador, mas no meu caso não foi.... vamos ao safe outra vez... e na segunda vez repetindo a gravação, lembrei de ter esquecido de mandar apagar as configs antigas no comando sysupgrade, usando a opção **-n**
```
> # sysupgrade -n -F openwrt-18.06.2-ar71xx-generic-archer-c5
-v1-squashfs-sysupgrade.bin
```
1. E se tudo deu certo, estamos prontos para iniciar o roteador e acessar ele por cabo conectado e pelo ip 192.168.1.1
```
ping 192.168.1.1
```
1. Agora com o firmware de antes [OEM](http://www.tplink.com/en/support/download/?model=Archer+C7) vamos cortar os primeiros 0x20200 (that is 131,584 = 257*512) Bytes do firmware original (1*512 Vendor-info + 256*512 U-Boot):
```
dd if=ArcherC5v1_en_3_14_3_up_boot\(150428\).bin of=tplink.bin skip=257 bs=512
31744+0 registros de entrada
31744+0 registros de saída
16252928 bytes (16 MB, 16 MiB) copiados, 0,116377 s, 140 MB/s
```
1. Agora podemos fazer a gravação direto pela interface da Luci em system - firmware ou usando o shell:
```
scp tplink.bin root@192.168.1.1:/tmp/
```
1. Conectar no router:
```
ssh root@192.168.1.1 -v
```
1. Agora ir a pasta tmp e rodar o procedimento mais seguro que é o sysupgrade
```
sysupgrade /tmp/tplink.bin
```
1. Ou menos aconselhado, mas eficiente mtd:
```
mtd -r write /tmp/tplink.bin firmware
```

E se tudo deu certo, voltamos a ter um roteador com a firmware de fábrica!

Happy Downgrading!!
