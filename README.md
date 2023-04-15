**Manual de criação de um servidor PXE para formatação de computadores através da rede usando [iPXE open source boot firmware](https://ipxe.org/).**

**A escolha do iPXE ao invés do pxelinux para o NBP (network boot program) foi devido a compatibilidade com certas BIOS/UEFI como de computadores da Lenovo.**

**Esse processo foi realizado no Ubuntu 20.04, 22.04 e debian 11 bullseye**

# 1. Instalando pacotes necessários

```
sudo apt install -y lighttpd dnsmasq nfs-kernel-server samba liblzma-dev git make gcc
```

|Pacote            |Uso                     |
|------------------|------------------------|
|lighttpd          |servidor HTTP/HTTPS     |
|dnsmasq           |servidor TFTP, DNS, DHCP|
|nfs-kernel-server |servidor NFS            |
|samba             |servidor SMB            |

os demais pacotes (liblzma-dev, git, gcc, make) serão utilizados para compilar o NBP (network boot program), nesse caso iPXE.

# 2. Criando a estrutua de pastas

**Crie a raiz do servidor TFTP e do servidor HTTP**
```
sudo mkdir /tftpboot
```

**Crie as subpastas ubuntu e windows para os arquivos desses sistemas operacionais**
```
sudo mkdir /tftpboot/{windows,ubuntu}
```

**Crie a pasta onde oos arquivos de instalação do windows ficarão**
```
sudo mkdir /mnt/windows
```

# 3. Compilando o NBP

**Clone o repositório**
```
git clone https://github.com/ipxe/ipxe.git
```

**Crie o *script* que vamos embarcar no NBP**
```
vim ipxe/src/embed.ipxe
```

**Esse script vai automatizar o acesso ao nosso menu de opções**
```
#!ipxe

isset ${next-server} || set next-server ${proxydhcp/dhcp-server}

dhcp
chain http://${next-server}/menu.ipxe || shell
```

**Vá até o subdiretório do projeto iPXE que clonamos**
```
cd ipxe/src
```

**Habilite o suporta ao NFS (necessário para Ubuntu no modo live)**
```
sed -i 's/#undef\tDOWNLOAD_PROTO_NFS/#define\tDOWNLOAD_PROTO_NFS/' config/general.h
```

**Habilite os comando ping, ipstat, reboot e poweroff respectivamente (opcional)**
```
sed -i 's/\/\/#define\ PING_CMD/#define\ PING_CMD/' config/general.h
```
```
sed -i 's/\/\/#define\ IPSTAT_CMD/#define\ IPSTAT_CMD/' config/general.h
```
```
sed -i 's/\/\/#define\ REBOOT_CMD/#define\ REBOOT_CMD/' config/general.h
```
```
sed -i 's/\/\/#define\ POWEROFF/#define\ POWEROFF/' config/general.h
```
Estes comandos são muito úteis caso o script não execute com sucesso.

**Compile o NBP (iPXE)**
``` 
make bin-x86_64-efi/ipxe.efi EMBED=embed.ipxe
```
Neste caso estamos compilando para sistemas com firmware UEFI e arquitetura 64 bits. caso queira compilar para outra arquitetura ou sistemas BIOS consulte a documentação do projeto: 
[build targets](https://ipxe.org/appnote/buildtargets)

**Copie o NBP para a pasta raiz do servidor TFTP**
```
sudo cp bin-x86_64-efi/ipxe.efi /tftpboot
```

# 4. Criando o menu de seleção

**Edite o menu de seleção na raiz do servidor TFTP**
```
sudo vim /tftpboot/menu.ipxe
```

```
#!ipxe

isset ${menu-default} || set menu-default WinPE

################################################## 
:start
menu Welcome to iPXE's Boot Menu
item WinPE Install Windows 10
item Ubuntu Ubuntu Live
item BootHardDisk Boot from Hard Disk
choose --default exit --timeout 15000 target && goto ${target}
################################################## 

:WinPE
  kernel http://${next-server}/windows/wimboot
  initrd http://${next-server}/windows/winpeshl.ini
  initrd http://${next-server}/windows/install.bat
  initrd http://${next-server}/windows/bootmgr.efi                  bootmgr.efi
  initrd http://${next-server}/windows/efi/boot/bootx64.efi         Bootx64.efi
  initrd http://${next-server}/windows/boot/bcd                     BCD
  initrd http://${next-server}/windows/boot/boot.sdi                boot.sdi
  initrd http://${next-server}/windows/sources/boot.wim             boot.wim
  boot

:Ubuntu
  kernel http://${next-server}/ubuntu/casper/vmlinuz
  initrd http://${next-server}/ubuntu/casper/initrd
  imgargs vmlinuz initrd=initrd root=/dev/nfs boot=casper file=preseed/ubuntu.seed keyboard-configuration/layoutcode=br netboot=nfs nfsroot=${next-server}:/tftpboot/ubuntu ip=dhcp --
  boot

:BootHardDisk
  exit
  goto start
```

# 5. Configurando o servidor TFTP e servidor DHCP (proxy)

**Edite o arquivo de configiração**
```
sudo vim /etc/dnsmasq.conf
```

**Adicione as seguintes entradas ao final do arquivo**
```
# Desabilita o servidor DNS embutido já que não precisamos dele
port=0

# Habilita o servidor TFTP
enable-tftp

# Configura a pasta raiz do nosso servidor TFTP
tftp-root=/tftpboot

# Faz com que as máquinas de firmware UEFI e arquitetura 64 bits recebam o arquivo ipxe.efi que compilamos anteriormente
pxe-service=x86-64_EFI,,ipxe.efi

# Faz com que as máquinas de firmware BIOS e arquitetura 32/64 bits recebam o arquivo ipxe.pxe não abordado nesse manual
# pxe-service=x86PC,,ipxe.pxe

# Configura o servidor DHCP no modo proxy, dessa maneira o dnsmasq não interfere com outros Servidores DHCP
# na sua rede e entrega apenas as informações necessárias para o boot.
dhcp-range=<endereço de sub-rede>,proxy

# Ativa o log para o servidor DHCP
log-dhcp

# Configura todas as entradas de log para um arquivo separado
log-facility=/var/log/dnsmasq.log
```
Troque **<endereço de sub-rede>** pela endereço da sua rede. Exemplo: 192.168.0.0

Caso a opção **pxe-service** não funcione consulte [PXE server](https://wiki.archlinux.org/title/dnsmasq#PXE_server).

**Reinicie o servidor TFTP/DHCP**
```
sudo systemctl restart dnsmasq.service
```

# 6. Configurando o servidor NFS

**Edite o arquivo /etc/exports**
```
sudo vim /etc/exports
```
**Adicione a entrada ao final do arquivo**
```
/tftpboot/ubuntu        <endereço de subrede>/<máscare de rede>(ro,no_root_squash,no_subtree_check)
```
Troque **<endereço de subrede>/<máscare de rede>** pela sua rede e máscara. Exemplo: 192.168.0.0/24 ou 192.168.0.0/255.255.255.0

**Exporte a pasta que configuramos no /etc/exports**
```
sudo exportfs -av
```

**Reinicie o servidor NFS**
```
sudo systemctl restart nfs-kernel-server.service
```

# 7. Configurando o servidor HTTP/HTTPS

**Modifique a pasta raiz do servidor HTTP/HTTPS do padrão /var/www/html para /tftpboot**
```
sudo sed -i 's/\/var\/www\/html/\/tftpboot/' /etc/lighttpd/lighttpd.conf
```

**Reinicie o servidor HTTP/HTTPS**
```
sudo systemctl restart lighttpd.service
```

# 8. Preparando os arquivos do Ubuntu

**Crie um diretório para montar a ISO**
```
sudo mkdir /mnt/cdrom
```

**Monte a ISO**
```
sudo mount <ubuntu iso path> /mnt/cdrom
```

**Copie todo o conteúdo da ISO para a pasta /tftpboot/ubuntu**
```
sudo cp -rv /mnt/cdrom/. /tftpboot/ubuntu
```

**Desmonte a ISO**
```
sudo umount /mnt/cdrom
```

**Configurando um arquivo preseed (opcional)**
```
sudo vim /tftpboot/ubuntu/preseed/ubuntu.seed
```
Caso este arquivo esteja como "somente leitura", execute:
```
sudo chmod 664 /tftpboot/ubuntu/preseed/ubuntu.seed
```


```
# Enable extras.ubuntu.com.
d-i     apt-setup/extras        boolean true
# Install the Ubuntu desktop.
tasksel tasksel/first   multiselect ubuntu-desktop
# On live DVDs, don't spend huge amounts of time removing substantial
# application packages pulled in by language packs. Given that we clearly
# have the space to include them on the DVD, they're useful and we might as
# well keep them installed.
ubiquity        ubiquity/keep-installed string icedtea6-plugin openoffice.org
d-i  base-installer/kernel/altmeta   string hwe-18.04

# The values can also be preseeded individually for greater flexibility.
#d-i debian-installer/language string en
#d-i debian-installer/country string NL
d-i debian-installer/locale string pt_BR.UTF-8
# Optionally specify additional locales to be generated.
#d-i localechooser/supported-locales multiselect en_US.UTF-8, nl_NL.UTF-8

# Keyboard selection.
# Disable automatic (interactive) keymap detection.
d-i console-setup/ask_detect boolean false
d-i keyboard-configuration/xkb-keymap select br
# To select a variant of the selected layout:
#d-i keyboard-configuration/xkb-keymap select us(dvorak)
# d-i keyboard-configuration/toggle select No toggling
```
Esse arquivo configura o teclado do Ubuntu live para ABNT2

# 9. Configurando o servidor SMB

**Edite o arquivo de configuração /etc/samba/smb.conf**
```
vim /etc/samba/smb.conf
```

**Adicione as seguintes entradas ao final do arquivo**
```
[windows]
	path = /mnt/windows
	guest ok = yes
	read only = no
```
isso compartilha a nossa pasta com os arquivos de instalação do windows que criamos anteriormente.

**Teste o nosso arquivo de configuração**
```
testparm
```
A saída abaixo informa que o arquivo não tem erros de sintaxe
```
Load smb config files from /etc/samba/smb.conf
Loaded services file OK.
```

**Reinicie o servidor SMB**
```
sudo systemctl restart smbd.service
```

# 10. Preparando os arquivos do Windows

**Monte a ISO do windows**
```
sudo mount <caminho da iso> /mnt/cdrom
```
Substitua <caminho da iso> pelo diretório onde a ISO está localizada. Exemplo: ~/windows10.iso

**Vá até o diretório onde montamos a ISO**
```
cd /mnt/cdrom
```

**Copie os arquivos necessários para iniciar o Windows para a pasta /tftpboot/windows**
```
sudo cp -v --parents {bootmgr,bootmgr.efi,boot/boot.sdi,boot/bcd,sources/boot.wim,efi/boot/bootx64.efi} /tftpboot/windows
```

**Copie os arquivos de instalação para a pasta /mnt/windows**
```
sudo rsync -av --exclude="sources/boot.wim" {setup.exe,sources} /mnt/windows
```

**Vá até a pasta /tftpboot/windows**
```
cd /tftpboot/windows
```

**Desmonte a ISO do windows**
```
sudo umount /mnt/cdrom
```

**Faça o Download do bootloader wimboot**
```
sudo wget 'https://github.com/ipxe/wimboot/releases/latest/download/wimboot'
```
Esse executável é necessário para iniciar imagens .wim como boot.wim e install.wim presentes na ISO do windows

**Crie o arquivo winpeshl.ini na pasta /tftpboot/windows**
```
sudo vim /tftpboot/windows/winpeshl.ini
```
Esse arquivo é um *script* que diz o que o windows executará ao iniciar o ambiente de pré execução

```
[LaunchApp]
install.bat
```
Estamos configurando ele para executar outro *script* chamado install.bat

**Crie o *script* install.bat**
```
sudo vim /tftpboot/windows/install.bat
```
O processo de executar esse script com máquinas BIOS 32/64 bits não funciona, é necessário embarcar o *script* na imagem boot.wim manualmente, consulte:
- [Windows WinPE](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-intro)
- [WinPE montar e customizar](https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/winpe-mount-and-customize)


```
wpeinit

net use i: \\<ip do servidor SMB>\windows

i:\setup.exe
```
Substitua **<ip do servidor SMB>** pelo endereço IP fixo da máquina onde fizemos todo esse processo. Esse *script* mapeia a pasta compartilhada /mnt/windows e executa o instalado do windows.
