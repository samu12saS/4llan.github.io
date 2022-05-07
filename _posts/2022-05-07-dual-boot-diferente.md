---
layout: post
title: Dual-boot diferente - Usando máquina virtual no Linux para dar boot em partição física com Windows 10
tags: libvirt kvm qemu windows linux dual-boot bitlocker
---

Multi-booting é a ação de instalar múltiplos sistemas operacionais num mesmo computador e poder escolher qual deles você quer iniciar (ou "dar o boot").

Uma das configurações mais populares de multi-boot é a inicialação dupla ("dual-boot") com Windows e Linux.

Para ter um sistema com dual-boot, normalmente se faz o seguinte:

* Instalação do Windows, particionando apenas uma parte do disco;
* Instalação do Linux no espaço livre deixado;
* Instalação de um bootloader (GRUB, por exemplo) para que seja possível escolher qual sistema operacional você deseja iniciar;

Lembrei que há mais de 10 anos eu tinha uma instalação dual-boot (Windows XP e Ubuntu) e, na época, acabei seguindo uma dica[^1] para que fosse possível dar boot na instalação física do Windows por meio de uma máquina virtual do VMware no Linux.

Só que as coisas (no meu computador principal) não são mais como eram antigamente:

|  | Windows | Linux | Hypervisor |
| --- | --- | --- | --- |
| Antigamente | XP | Ubuntu | VMware Workstation |
| "Presentemente" | 10 | Fedora | KVM |

Mas a vontade de dar boot na partição física do Windows em uma máquina virtual no Linux continuou :)

Pesquisando sobre o assunto, confirmei que é possível dar boot numa instalação física do Windows usando uma máquina virtual do QEMU.[^2] [^3]

Em vez de passar o disco inteiro para a máquina virtual tendo o risco de corromper as outras partições e seus dados, é recomendado criar dispositivos loopback para a partição EFI e tabela GPT. Depois, basta criar uma RAID-0 virtual com estes dispositivos e a partição física do Windows.

## Preparação do disco virtual

Usando o esquema GPT, vamos criar dois arquivos para conter a partição EFI e um espaço vazio:
 - `efi0` de 100MB
 - `efi1` de 1MB

```
dd if=/dev/zero of=efi0 bs=1M count=100
dd if=/dev/zero of=efi1 bs=1M count=1
```

Com os dois arquivos criados, precisamos transformá-los em dispositivos de bloco (loopback devices) usando o `losetup`:

```
sudo losetup loop0 efi0
sudo losetup loop1 efi1
```

Se a criação dos dispositivos deu certo, eles serão listados no comando:

```
losetup -a
```

Depois, é preciso juntar os dispositivos loopack e a partição do Windows numa RAID-0 virtual na seguinte sequência:
 - /dev/loop0
 - /dev/[partição do Windows - no meu caso, sda4]
 - /dev/loop1

 ```
 sudo mdadm --build --verbose /dev/md0 --chunk=512 --level=linear --raid-devices=3 /dev/loop0 /dev/sda4 /dev/loop1
 ```

Temos então um disco virtual "zerado" contendo a partição física do Windows. Precisamos então criar a tabela de partições deste disco:

```
% sudo parted /dev/md0
GNU Parted 3.4
Using /dev/md0
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) unit s
(parted) mktable gpt
(parted) mkpart primary fat32 2048 204799
(parted) mkpart primary ntfs 204800 -2049
(parted) set 1 boot on
(parted) set 1 esp on
(parted) set 2 msftdata on
(parted) name 1 EFI
(parted) name 2 Windows
(parted) quit
```

 * `unit s` define a unidade padrão a ser usada para _setores_
 * cada setor tem 512 bytes
 * 204800 setores = 102400 KiB = 100 MiB

A tabela de partições do RAID-0 ficará assim:
 - `/dev/md0p1` - partição EFI
 - `/dev/md0p2` - partição física do Windows

O próximo passo é formatar a partição EFI como um sistema de arquivos FAT32:

```
sudo mkfs.msdos -F 32 -n EFI /dev/md0p1
```

Crie uma máquina virtual usando o _virt-manager_ com os padrões para o Windows 10 e adicione o disco virtual `/dev/md0` como disco SATA.

![Máquina virtual - disco /dev/md0](/assets/dual-boot-1.png)

Agora temos que recuperar os arquivos de boot do Windows usando o **BCDBoot**[^4].

Para poder usar esta ferramenta é preciso dar boot na mídia de instalação do Windows. Quando aparecer a janela de instalação, pressione **Shift + F10** para abrir uma janela do prompt de comando.

![Janela de instalação do Windows 10](/assets/dual-boot-2.png)
![Shift + F10 para abrir a janela do prompt de comando](/assets/dual-boot-3.png)

Antes, devemos atribuir uma letra para a partição EFI usando o **diskpart**[^5].

Selecione o disco e a partição corretas para que `S:` aponte para a partição EFI.

```
X:\Sources>diskpart
DISKPART> list disk
DISKPART> select disk 0
DISKPART> list volume
DISKPART> select volume 2
DISKPART> assign letter=S
DISKPART> exit
```

E no meu caso a unidade `C:` estava com o BitLocker ativado. Tive que desbloqueá-la usando o **manage-bde**[^6] antes de recuperar os arquivos de boot do Windows pelo **BCDBoot**.

![Desbloqueando o Bitlocker usando o manage-bde](/assets/dual-boot-4.png)

Por fim, criamos os arquivos de configuração de boot do Windows:

```
bcdboot C:\Windows /s S: /f ALL
```

Pronto! Agora basta reiniciar a máquina virtual que o Windows da partição física será iniciado com sucesso.

![Máquina virtual iniciando o Windows 10 a partir da partição física](/assets/dual-boot-5.png)

## Observações finais

Toda vez que reiniciar o Linux será preciso recriar os dispositivos de loopback (loop0 e loop1) e o disco virtual (md0).

A máquina virtual, por padrão, não terá uma GPU dedicada. Não cheguei a testar _GPU passthrough_ ou _iGVT-g_ nela. O driver de vídeo QXL foi suficiente para minha necessidade.

É recomendável instalar o **SPICE guest tools**[^7] e o **Virtio-win guest tools**[^8] no Windows 10.

Se a unidade `C:` estiver criptografada, é bom adicionar uma unidade USB extra na VM e criar uma chave externa do BitLocker. A minha instalação do Windows estava com o BitLocker ativado com o _Trusted Module Platform_ (TPM). Como eu não quis mexer com _TPM passthrough_, achei mais simples criar a chave externa e deixá-la num disco QCOW2 como sendo uma unidade USB.

BCD significa "Boot Configuration Data".

---

## virsh dumpxml win10-dual-boot

```xml
<domain type='kvm'>
  <name>win10-dual-boot</name>
  <uuid>00000000-1111-2222-3333-444444444444</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>2</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-6.1'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/edk2/ovmf/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win10-dual-boot_VARS.fd</nvram>
    <bootmenu enable='no'/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vpindex state='on'/>
      <synic state='on'/>
      <stimer state='on'/>
      <reset state='on'/>
    </hyperv>
    <vmport state='off'/>
  </features>
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' cores='2' threads='1'/>
  </cpu>
  <clock offset='localtime'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='hypervclock' present='yes'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/iso/pt_windows_10_enterprise_ltsc_2019_x64_dvd_d43dcbad.iso'/>
      <target dev='sda' bus='sata'/>
      <readonly/>
      <boot order='1'/>
      <address type='drive' controller='0' bus='0' target='0' unit='0'/>
    </disk>
    <disk type='block' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native'/>
      <source dev='/dev/md0'/>
      <target dev='sdb' bus='sata'/>
      <boot order='2'/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/win10-usb.qcow2'/>
      <target dev='sdc' bus='usb' removable='on'/>
      <readonly/>
      <address type='usb' bus='0' port='4'/>
    </disk>
    <controller type='usb' index='0' model='qemu-xhci' ports='15'>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
    </controller>
    <controller type='sata' index='0'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1f' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pcie-root'/>
    <controller type='pci' index='1' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='1' port='0x10'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0' multifunction='on'/>
    </controller>
    <controller type='pci' index='2' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='2' port='0x11'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x1'/>
    </controller>
    <controller type='pci' index='3' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='3' port='0x12'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x2'/>
    </controller>
    <controller type='pci' index='4' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='4' port='0x13'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x3'/>
    </controller>
    <controller type='pci' index='5' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='5' port='0x14'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x4'/>
    </controller>
    <controller type='virtio-serial' index='0'>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
    </controller>
    <interface type='network'>
      <mac address='52:54:00:11:22:33'/>
      <source network='default'/>
      <model type='virtio'/>
      <link state='up'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </interface>
    <serial type='pty'>
      <target type='isa-serial' port='0'>
        <model name='isa-serial'/>
      </target>
    </serial>
    <console type='pty'>
      <target type='serial' port='0'/>
    </console>
    <channel type='spicevmc'>
      <target type='virtio' name='com.redhat.spice.0'/>
      <address type='virtio-serial' controller='0' bus='0' port='1'/>
    </channel>
    <input type='tablet' bus='usb'>
      <address type='usb' bus='0' port='1'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='spice' autoport='yes'>
      <listen type='address'/>
      <image compression='off'/>
      <gl enable='no'/>
    </graphics>
    <sound model='ich9'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1b' function='0x0'/>
    </sound>
    <audio id='1' type='spice'/>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
    </video>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='2'/>
    </redirdev>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='3'/>
    </redirdev>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </memballoon>
  </devices>
</domain>
```

---

[^1]: [Never reboot linux again?  Run your existing Windows install in Linux! \| Mohammad Azimi](https://mazimi.wordpress.com/2007/06/24/virtualization-of-an-existing-physical-partition-of-windows-within-linux/)

[^2]: [QEMU - ArchWiki - Using any real partition as the single primary partition of a hard disk image](https://wiki.archlinux.org/title/QEMU#Using_any_real_partition_as_the_single_primary_partition_of_a_hard_disk_image)

[^3]: [Boot physical Windows inside Qemu guest machine \| Moez Bouhlel [lejenome] Website](https://lejenome.tik.tn/post/boot-physical-windows-inside-qemu-guest-machine)

[^4]: [BCDBoot Command-Line Options \| Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/bcdboot-command-line-options-techref-di?view=windows-10)

[^5]: [diskpart \| Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/diskpart)

[^6]: [manage-bde \| Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/manage-bde)

[^7]: [SPICE - Download - Guest - Windows binaries](https://www.spice-space.org/download.html#windows-binaries)

[^8]: [WindowsGuestDrivers/Download Drivers - KVM](https://www.linux-kvm.org/page/WindowsGuestDrivers/Download_Drivers)
