---
title: "Compilar TP-Link CPE 210 v2&3"
date: 2019-02-01
tags: [generic, mesh, openwrt, LiMe, ]
header:
  # image: "/images/8bitheart.jpg"
# excerpt: "FLOSS, OpenWRT, Mesh, InfoSec"
---
Para usar o LiMe em CPE 210 v3 ainda nao temos opção no chef (março 2019), entao precisamos compilar ele partindo do openwrt, para facilitar acabamos  escolhendo uma versao com o port do ChipSet mt7620 pronto para evitar ter que aplicar patches na compilação.

Este passo a passo esta baseado no que o  @HiureAnderson fez, mas escrito em markdown e com adição de imagens e alguns pacotes, orginal em :

https://github.com/hiureanderson/cadernos-zim/blob/master/compilando%20Cpe210-v3/Home.txt

Após descer o ambiente de build e testarmos o mesmo por padrão, ou seja compilar uma imagem "normal" do openwrt para a cpev3, precisamos adicionar os feeds do LiMe, escolhendo os pacotes e meta pacotes certos, removendo os conflitantes e adicionando a parte de ambiente gráfico.

## Usar ambiente pronto

Para facilitar nossa vida, vamos descer o fork de @robimarko do git que ja esta com os patches aplicados,  prestando atenção ao branch especifico apontado no clone:

	# git clone -b CPE210-v3-PR --single-branch https://github.com/robimarko/openwrt.git

renomear o repositorio pra lembrar que eh especifico com o v3

	# mv openwrt/ openwrtCPEV3/

### teste ambiente, antes do Lime prorpiamente dito

Entrar no diretório pra iniciar a compilação e testar compilador padrão sem o Lime:

	 # make menuconfig

```
Target: AR7xxx/AR9xxx

Subtarget: Generic

Target Profile: TP-LINK CPE210 V3
```

após configurados esses passos, vamos gerar a primeira imagem, nesse caso usando o comando time para saber quanto demorou o processo e a opção -j9 para usar todos os cores de minha maquina, 8 neste caso, valendo a regra que para uso total dos processadores a opção j deve ser todos procs+1:

	# make download

	# time make -j9 V=s

a saída da compilação deve apresentar um log apontando onde esta a imagem e o tempo:

```
real	39m9,072s
user	222m16,105s
sys	18m41,099s
```

agora para conferir a imagem...

	# cd bin/targets/ar71xx/generic/

	# ls -alFh

```
-rw-r--r-- 1 rtroian rtroian 3,3M jan 12 16:15 openwrt-ar71xx-generic-cpe210-v3-squashfs-factory.bin
-rw-r--r-- 1 rtroian rtroian 7,5M jan 12 16:15 openwrt-ar71xx-generic-cpe210-v3-squashfs-sysupgrade.bin
```

os tamanhos e as funções parecem ok, agora vamos movimentar a imagem para outra pasta pois a nova compilação vai sobrescrever os arquivos gerados:

	# mv  bin/targets/ar71xx/generic/* /home/rT/firmwares/openwrt/cpe210v3

### Gravar OpenWRt puro

agora podemos gravar a imagem no roteador, substituindo a original da tplink! Em alguns casos pode ser necessário trocar o nome do arquivo factory para tplink.bin para que seja aceito.


### Limpar ambiente
se tudo deu certo, agora poderiamos limpar o ambiente de compilação, mas em nosso caso podemos testar não limpar por ser a mesma target, mas para registro o processo  seria:

	# make clean

	# make dist-clean


### Iniciar com o Lime

Agora vamos adicionar feeds do LiMe:

	 cp feeds.conf.default feeds.conf.default.local

	 cp feeds.conf.default feeds.conf

	  echo "src-git libremesh https://github.com/libremesh/lime-packages.git" >> feeds.conf
	 echo "src-git libremap https://github.com/libremap/libremap-agent-openwrt.git" >> feeds.conf
	 echo "src-git limeui https://github.com/libremesh/lime-packages-ui.git" >> feeds.conf

atualizar lista de pacotes dos feeds

	# scripts/feeds update -a

	# scripts/feeds install -a


### Compilar o Lime
Finalmente vamos compilar com o Lime, mas para isso precisamos ativar o software que  apontamos nos feeds e desativar alguns outros:

entramos na configuração  ncurses por terminal:

	 # make menuconfig

e nas opções de pacotes, temos as seguintes necessidades do Lime:

	Base system : desativa  dnsmasq e selecionar dnsmasq-dhcpv6
	Network: desativar odhcpd
 	Lime-collections: ativar lime-full

E ao compilar temos uma versão funcionando na malha, mas espere, ainda não acabamos, por algum motivo essas seleções de pacotes dão problemas no ambiente grafico, então temos que ir conferir a Luci e a seção admin

	Luci: luci-base e luci-mod-failsafe, modules - admin full

E agora sim, ao sair e salvar temos  uma provavel imagem 100% funcional!

	time make -j9 V=s

Se tudo deu certo, ao listar a pasta (lembrando de ter removido o anterior, ou conferindo a hora da criação da imagem) ao final devera ter isso:

	# ls bin/targets/ar71xx/generic/ -lah

```
total 30M
drwxr-xr-x 3 rtroian rtroian 4,0K jan 13 09:22 .
drwxr-xr-x 3 rtroian rtroian 4,0K jan 13 09:18 ..
-rw-r--r-- 1 rtroian rtroian 4,4K jan 24 13:01 config.seed
-rw-r--r-- 1 rtroian rtroian 5,1M jan 24 13:04 openwrt-ar71xx-generic-cpe210-v3-squashfs-factory.bin
-rw-r--r-- 1 rtroian rtroian 7,5M jan 24 13:04 openwrt-ar71xx-generic-cpe210-v3-squashfs-sysupgrade.bin
-rw-r--r-- 1 rtroian rtroian 5,8K jan 24 13:04 openwrt-ar71xx-generic-device-cpe210-v3.manifest
```

Depois o resto do processo é o mesmo de gravação padrão!

Eras isso pessoal,

Happy Meshing
