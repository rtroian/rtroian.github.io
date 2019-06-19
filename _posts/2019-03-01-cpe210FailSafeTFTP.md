---
title: "TP-Link CPE 210 FailSafe e TFTP"
date: 2019-03-01
tags: [generic, mesh, openwrt, LiMe, ]
header:
  # image: "/images/8bitheart.jpg"
# excerpt: "FLOSS, OpenWRT, Mesh, InfoSec"
---
Em caso de problema com alguma gravação no cpe, precisaremos usar o modo de failsafe para executar nova gravação e restaurar o sistema e para isso precisamos seguir os seguintes  passos:

 1- Para entrar em failsafe, desligar o roteador pelo cabo de rede, contar 5 segundos, colocar um pino em reset, ligar contando até 10,  o led da lan deve ligar sem ter nada conectado nela...


2 - Assim que ela ligar colocar cabo da lan no roteador e na maquina, colocar a maquina local em ip 192.168.0.100/24 pois o roteador está por padrão de fábrica em 192.168.0.254. Isso pode ser feito de distintas maneiras, via terminal é aconselhado desligar o gerenciador de rede e após isso fixar o ip.

	$ systemctl stop NetworkManager

e conferir com

	$ systemctl status NetworkManager
```
	● NetworkManager.service - Network Manager
   	Loaded: loaded (/usr/lib/systemd/system/NetworkManager.service; enabled; vendor preset: disabled)
 	 Drop-In: /usr/lib/systemd/system/NetworkManager.service.d
           └─NetworkManager-ovs.conf
  	 Active: inactive (dead) since Wed 2019-01-23 18:06:42 -02; 8s ago
```
após isso adicionar ip na interface:

	$ ip a add 192.168.0.100/24 dev enp5s0


 3- Conferir com tcpdump apontando para a interface local

	$ tcpdump -nli enp5s0

```
1280 ARP, Request who-has 192.168.0.254 tell 192.168.0.100, length 28
18:16:31.181334 ARP, Reply 192.168.0.254 is-at ba:be:fa:ce:08:41, length 46
18:16:32.462956 IP 192.168.0.254.3422 > 192.168.0.100.69:  31 RRQ "recovery.bin" octet timeout 2
18:16:32.463009 IP 192.168.0.100 > 192.168.0.254: ICMP 192.168.0.100 udp port 69 unreachable, length 67
18:16:35.663035 IP 192.168.0.254.3422 > 192.168.0.100.69:  31 RRQ "recovery.bin" octet timeout 2
18:16:35.663100 IP 192.168.0.100 > 192.168.0.254: ICMP 192.168.0.100 udp port 69 unreachable, length 67
```

4 -  O failsafe esta pronto para pedir um arquivo por tftp para a maquina .100, por isso colocamos o ip no passo anterior,  agora precisamos subir um servidor local e depois colocar o arquivo solicitado em uma pasta que esse servidor utilize . Para isso criamos a pasta /tftpboot com permissão geral e dono nobody:

	$ mkdir /tftpboot
	$ chmod 777 /tftpboot
	$ chown nobody /tftpboot

Nesse tcpdump acima também  vemos que o roteador esta pedindo um arquivo chamado recovery.bin, que será usado nos próximos passos, mas antes vamos iniciar o serviço de tftp em nossa maquina:

	 # atftpd --daemon --bind-address 192.168.0.100 /tftpboot/

e conferir os  logs do journalctl:

	journactl -f
	...
	jan 23 18:17:38 Darth atftpd[5947]: Advanced Trivial FTP server started (0.7.1)
	jan 23 18:17:38 Darth atftpd[5948]: atftpd: can't bind port 192.168.0.100:69/udp

como user normal não  deu certo, tentar como root:

	$ atftpd --daemon --bind-address 192.168.0.100 /tftpboot/

e os logs:

	jan 23 18:19:27 Darth atftpd[5979]: Advanced Trivial FTP server started (0.7.1)
	jan 23 18:19:27 Darth sudo[5978]: pam_unix(sudo:session): session closed for user root
	jan 23 18:19:27 Darth atftpd[5980]: atftpd: can't change identity to nobody.nogroup, exiting.


conferindo no sistema, vi que /etc/group não existia o grupo  "nogroup",  então para isso foi criado com o comando:

	$ groupadd nogroup

e acompanhando o terminal de  logs por

	# journalctl -F

vemos que o servidor iniciou sem problemas agora!

```
: Advanced Trivial FTP server started (0.7.1)
jan 23 18:26:03 Darth sudo[6244]: pam_unix(sudo:session): session closed for user root
jan 23 18:26:03 Darth atftpd[6248]: Serving recovery.bin to 192.168.0.254:3487
```

6 - Agora que o servidor local ja reconhece a pasta e o roteador como cliente, precisamos identificar o que o roteador está solicitando através do terminal do tcpdump:

```
18:28:46.456018 IP 192.168.0.254.1842 > 192.168.0.100.69:  31 RRQ "recovery.bin" octet timeout 2
18:28:46.456311 IP 192.168.0.100.52866 > 192.168.0.254.1842: UDP, length 19
```
o tcpdump esta mostrando que a requisição (RRQ) do arquivo esta dando timeout, o que siginifica que não está achando o arquivo chamado recovery.bin para servir.

7 - Finalmente, precisamos copiar para a pasta  /tftpboot o firmware a ser utilizado, prestando atenção que seja a  **versao factory**, e neste processo renomear para recovery.bin:

	mv /home/rt/firmwareoriginal.bin /tftpboot/recovery.bin

e acompanhar pelo terminal do tcpdump o inicio ao fim da troca de dados.

```
18:42:34.859906 ARP, Request who-has 192.168.0.100 tell 192.168.0.254, length 46
18:42:34.859945 ARP, Reply 192.168.0.100 is-at 28:d2:44:2c:31:53, length 28
18:42:34.859998 IP 192.168.0.254.1878 > 192.168.0.100.69:  31 RRQ "recovery.bin" octet timeout 2
18:42:34.860449 IP 192.168.0.100.33171 > 192.168.0.254.1878: UDP, length 12
18:42:34.860516 IP 192.168.0.254.1878 > 192.168.0.100.33171: UDP, length 4
18:42:34.860675 IP 192.168.0.100.33171 > 192.168.0.254.1878: UDP, length 516
```

E agora é só  ver o fluir dos pacotes na rede... quando os pacotes param de apresentar timeout e iniciam a troca entre maquina e router significa que a imagem esta se movimentando para o router e assim que ela acabar  o roteador vai reiniciar.

Agora para sabermos se a imagem está se comportando como deveria precisamos  reativar o network manager e receber ip via cabo lan!

	$ systemctl start NetworkManager
	$ dhclient enp5s0 -v

Se tudo isso deu certo, temos uma versão funcional de cpe/roteador novamente!

happy debricking!
rT

v0 em 24/01
