---
layout: post
title: GPU passthrough via OVMF e VFIO - Passando uma placa de vídeo para máquina virtual
tags: libvirt kvm gpu-passthrough ovmf vfio
---

O caso descrito neste post considera o *HP EliteDesk 805 G6 Small Form Factor PC* com as seguintes especificações:
 - Processador: AMD Ryzen 7 PRO 4750G
 - GPU integrado: AMD Radeon Vega 7 Graphics
 - GPU offboard: AMD Radeon R7 430 2 GB GDDR5 64-bit 2 DP Graphics Card

## Pré-requisitos
 - Ter pelo menos duas GPUs;
   - Sugestão: onboard para o host e offboard para a máquina virtual;
 - IOMMU ativo;
 - Máquina virtual no libvirt com UEFI (OVMF);
 - Opcionalmente, ter um monitor plugado na saída da GPU que será utilizada para a máquina virtual;

O script a seguir verifica se o IOMMU está ativo e com os grupos válidos:

```bash
#!/bin/bash
shopt -s nullglob
for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

## Isolando a GPU

Identifique a GPU que será isolada por meio do comando `lspci`.

```
$ lspci -D | grep VGA
0000:01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Oland [Radeon HD 8570 / R5 430 OEM / R7 240/340 / Radeon 520 OEM] (rev 87)
0000:07:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Renoir (rev d8)
```

No meu caso, `0000:01:00.0` é o ID da placa offboard.

### modprobe.d

Utilize o `modprobe` para inicializar o módulo `vfio-pci` durante o boot. Todos os dispositivos do grupo IOMMU precisam ser informados para o `vfio-pci`.

```
$ lspci -n -s 01:00
01:00.0 0300: 1002:6611 (rev 87)
01:00.1 0403: 1002:aab0
```

Crie o arquivo `/etc/modprobe.d/vfio.conf` adicionando os identificadores dos dispositivos:
```
options vfio-pci ids=1002:6611,1002:aab0
```

Para descobrir o driver atualmente em uso pela GPU que será isolada, use o comando:
```
$ lspci -v -s 01:00
01:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Oland [Radeon HD 8570 / R5 430 OEM / R7 240/340 / Radeon 520 OEM] (rev 87) (prog-if 00 [VGA controller])
        Subsystem: Hewlett-Packard Company Device 3375
        Flags: bus master, fast devsel, latency 0, IRQ 97, IOMMU group 0
        Memory at c0000000 (64-bit, prefetchable) [size=256M]
        Memory at fc800000 (64-bit, non-prefetchable) [size=256K]
        I/O ports at 3000 [size=256]
        Expansion ROM at fc860000 [disabled] [size=128K]
        Capabilities: <access denied>
        Kernel driver in use: vfio-pci
        Kernel modules: radeon, amdgpu

01:00.1 Audio device: Advanced Micro Devices, Inc. [AMD/ATI] Oland/Hainan/Cape Verde/Pitcairn HDMI Audio [Radeon HD 7000 Series]
        Subsystem: Hewlett-Packard Company Device aab0
        Flags: bus master, fast devsel, latency 0, IRQ 94, IOMMU group 0
        Memory at fc840000 (64-bit, non-prefetchable) [size=16K]
        Capabilities: <access denied>
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
```

Crie o arquivo `/etc/modprobe.d/blacklist.conf` e adicione a diretiva `blacklist` juntamente com o driver retornado, para que o mesmo não seja carregado pelo host (afinal, o dispositivo será passado inteiramente para a máquina virtual):
```
blacklist radeon
```

Gere uma nova imagem do `initramfs` após as modificações nas configurações do `modprobe`.

## Configuração da máquina virtual

Crie uma nova máquina virtual no `virt-manager` com as seguintes especificações:
 - Sistema Operacional: Microsoft Windows 10
 - Firmware: UEFI (OVMF)
 - Modelo de CPU: `host-passthrough`
 - Modelo de vídeo: QXL

Edite manualmente a configuração XML da máquina virtual (`virsh edit`) e adicione a entrada `vendor_id` da feature `hyperv` com um valor qualquer:

```xml
    <hyperv>
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="on"/>
      <vendor_id state="on" value="00110011"/>
    </hyperv>
```

Depois que instalar o Windows 10, adicione todos os dispositivos PCI do grupo IOMMU na máquina virtual.

![vm-pci-devs.png](/assets/vm-pci-devs.png)

### virsh dumpxml win10-uefi-clone
```xml
<domain type='kvm'>
  <name>win10-uefi-clone</name>
  <uuid>00000000-1111-2222-3333-444444444444</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>4</vcpu>
  <os>
    <type arch='x86_64' machine='pc-q35-6.1'>hvm</type>
    <loader readonly='yes' type='pflash'>/usr/share/edk2/ovmf/OVMF_CODE.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/win10-uefi-clone_VARS.fd</nvram>
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
      <vendor_id state='on' value='00110011'/>
    </hyperv>
    <vmport state='off'/>
  </features>
  <cpu mode='host-passthrough' check='partial' migratable='on'>
    <topology sockets='1' dies='1' cores='4' threads='1'/>
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
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/hdd/win10-uefi-clone.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <boot order='1'/>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/iso/pt_windows_10_enterprise_ltsc_2019_x64_dvd_d43dcbad.iso'/>
      <target dev='sdb' bus='sata'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='1'/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/var/lib/libvirt/images/iso/virtio-win-0.1.204.iso'/>
      <target dev='sdc' bus='sata'/>
      <readonly/>
      <address type='drive' controller='0' bus='0' target='0' unit='2'/>
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
    <controller type='pci' index='6' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='6' port='0x15'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x5'/>
    </controller>
    <controller type='pci' index='7' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='7' port='0x16'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x6'/>
    </controller>
    <controller type='pci' index='8' model='pcie-root-port'>
      <model name='pcie-root-port'/>
      <target chassis='8' port='0x17'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x7'/>
    </controller>
    <controller type='virtio-serial' index='0'>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
    </controller>
    <interface type='network'>
      <mac address='52:54:00:11:33:55'/>
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
    </graphics>
    <sound model='ich9'>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x1b' function='0x0'/>
    </sound>
    <audio id='1' type='spice'/>
    <video>
      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
    </video>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
    </hostdev>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x07' slot='0x00' function='0x0'/>
    </hostdev>
    <hostdev mode='subsystem' type='pci' managed='yes'>
      <source>
        <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
      </source>
      <address type='pci' domain='0x0000' bus='0x08' slot='0x00' function='0x0'/>
    </hostdev>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='2'/>
    </redirdev>
    <redirdev bus='usb' type='spicevmc'>
      <address type='usb' bus='0' port='3'/>
    </redirdev>
    <memballoon model='virtio'>
      <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
    </memballoon>
  </devices>
</domain>
```

Ligue a máquina virtual, instale os drivers de vídeo da GPU redirecionada ou aguarde o Windows instalá-los automaticamente e desligue a máquina novamente.

No `virt-manager`, remova o dispositivo gráfico e altere o modelo de vídeo para `none`. Depois, substitua o teclado e mouse por dois dispositivos `evdev` relativos aos que estão em uso no host.

```patch
--- win10-uefi-clone_qxl.xml	2022-06-27 09:38:44.316924953 -0300
+++ win10-uefi-clone_radeon.xml	2022-06-27 09:38:17.964727495 -0300
@@ -138,22 +138,20 @@
       <target type='virtio' name='com.redhat.spice.0'/>
       <address type='virtio-serial' controller='0' bus='0' port='1'/>
     </channel>
-    <input type='tablet' bus='usb'>
-      <address type='usb' bus='0' port='1'/>
+    <input type='evdev'>
+      <source dev='/dev/input/by-id/usb-PixArt_HP_320M_USB_Optical_Mouse-event-mouse'/>
+    </input>
+    <input type='evdev'>
+      <source dev='/dev/input/by-id/usb-Primax_HP_Wired_Desktop_320K_Keyboard-event-kbd' grab='all' repeat='on'/>
     </input>
     <input type='mouse' bus='ps2'/>
     <input type='keyboard' bus='ps2'/>
-    <graphics type='spice' autoport='yes'>
-      <listen type='address'/>
-      <image compression='off'/>
-    </graphics>
     <sound model='ich9'>
       <address type='pci' domain='0x0000' bus='0x00' slot='0x1b' function='0x0'/>
     </sound>
     <audio id='1' type='spice'/>
     <video>
-      <model type='qxl' ram='65536' vram='65536' vgamem='16384' heads='1' primary='yes'/>
-      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x0'/>
+      <model type='none'/>
     </video>
     <hostdev mode='subsystem' type='pci' managed='yes'>
       <source>
```

Ao ligar a máquina virtual, duas coisas acontecerão: (1) o mouse e o teclado ficarão congelados no host. Isso se deve aos dispositivos `evdev` adicionados à vm. Para alternar o controle destes dispositivos entre host e guest, pressione [`left Ctrl + right Ctrl`]; (2) A tela da máquina virtual ficará disponível na saída de vídeo que foi redirecionada (no caso deste post, na saída da placa offboard).

## Referências

[OVMF - KVM](https://www.linux-kvm.org/page/OVMF)

[VFIO - “Virtual Function I/O” &mdash; The Linux Kernel  documentation](https://docs.kernel.org/driver-api/vfio.html)

[PCI passthrough via OVMF - ArchWiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)

[Pci passthrough - Proxmox VE](https://pve.proxmox.com/wiki/Pci_passthrough)
