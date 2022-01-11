---
layout: post
title: Consumo anormal de CPU em máquinas virtuais QEMU com usb-tablet (USB 2)
tags: libvirt kvm qemu windows
---

Testando as instalações de máquinas virtuais no libvirt (KVM/QEMU), me deparei com um consumo anormal de CPU no Windows XP e Windows 7, mesmo com o sistema ocioso. "Anormal" para mim, neste caso, era mais ou menos 10% reportado pelo libvirt, enquanto que pelo sistema operacional convidado era 1%.

![Windows XP - Consumo de CPU reportado pelo libvirt](/assets/libvirt-winxp-1.png)
![Windows XP - Consumo de CPU reportado pelo Windows](/assets/libvirt-winxp-2.png)
![Windows 7 - Consumo de CPU reportado pelo libvirt](/assets/libvirt-win7-1.png)
![Windows 7 - Consumo de CPU reportado pelo Windows](/assets/libvirt-win7-2.png)

Revisando as configurações das máquinas, resolvi mudar o controlador USB de `USB 2` para `USB 3`. O resultado inicial foi que o ponteiro do mouse deixou de funcionar de forma suave (passou a funcionar no modo relativo, não-absoluto), mas o consumo de CPU voltou ao nível esperado.

![Windows XP - Consumo de CPU normalizado após mudança no controlador USB](/assets/libvirt-winxp-3.png)
![Windows 7 - Consumo de CPU normalizado após mudança no controlador USB](/assets/libvirt-win7-3.png)

# `usb-tablet` + USB 2

[Em 2014 o Gerd Hoffmann fez um post explicando esta anormalidade quando se usa o dispositivo `usb-tablet` num controlador USB 2.](https://www.kraxel.org/blog/2014/03/qemu-and-usb-tablet-cpu-consumtion/)

Para contornar a situação, é preciso mudar o controlador USB para um que tenha suporte ao [xHCI](https://en.wikipedia.org/wiki/Extensible_Host_Controller_Interface) (eXtensible Host Controller Interface, ou simplesmente "USB 3"). Isso explica porque minha mudança de início deu certo. Mas o Windows só oferece suporte nativo ao xHCI a partir do Windows 8, e não existem drivers disponíveis do controlador USB 3 padrão do QEMU, modelo `qemu-xhci` (ID de hardware `PCI\VEN_1B36\DEV_000D`), para Windows XP e Windows 7.

![Windows XP - Controlador USB sem driver reconhecido](/assets/libvirt-winxp-4.png)
![Windows 7 - Controlador USB sem driver reconhecido](/assets/libvirt-win7-4.png)

## SPICE guest tools

O ponteiro do mouse volta a ter um comportamento normal instalando o [SPICE guest tools](https://www.spice-space.org/download.html#windows-binaries), mesmo sem um dispositivo `usb-tablet` funcional.

Portanto, se você não vai precisar de dispositivos USB emulados ou redirecionados, aqui jaz o problema.

# `nec-xhci`

Se você for utilizar dispositivos USB, a boa notícia é que o QEMU pode emular um controlador xHCI no qual existem drivers disponíveis para o Windows XP e Windows 7: o modelo `nec-xhci`, ou **NEC Electronics (Renesas) USB 3.0 Host Controller** (ID de hardware `PCI\VEN_1033\DEV_0194`).

Para adicioná-lo à máquina virtual, mude o atributo do modelo de controlador USB para `nec-xhci`. Usando o _virt-manager_, no momento em que você aplicar a mudança ele exibirá `USB 3` no campo, mas internamente esta propriedade foi alterada para `nec-xhci`.

![virt-manager - Mudando o modelo do controlador USB para nec-xhci](/assets/libvirt-nec-xhci-1.png)
![virt-manager - Controlador USB alterado internamente para nec-xhci](/assets/libvirt-nec-xhci-2.png)

## Download dos drivers para o `nec-xhci`

O site da Dell tem uma página de download para o driver deste controlador: https://www.dell.com/support/home/pt-br/drivers/driversdetails?driverid=36x7d

|  | [DRVR_Chipset_NEC_USB3_A02-36X7D_setup_ZPE.exe](https://dl.dell.com/FOLDER00364132M/12/DRVR_Chipset_NEC_USB3_A02-36X7D_setup_ZPE.exe) |
--- | --- |
| MD5 | c96dfefc2018ca1cc8f0f194ce5c386c |
| SHA1 | 12bfa3ff98753ec49cf477745ab993f4d969d2dd |
| SHA-256 | 6f7efc66f196c4c0cdc8121194c6ba2542cd0a4134f131fec33a5cf02827e724 |

E existe uma outra versão deste driver que aparentemente era distribuída pela Intel, de acordo com o pacote [renesas-usb-3-0-driver](https://community.chocolatey.org/packages/renesas-usb-3-0-driver) do Chocolatey.

|  | [USB3.0_allOS_2.1.28.1_PV.exe](https://downloadmirror.intel.com/19880/eng/USB3.0_allOS_2.1.28.1_PV.exe) |
| --- | --- |
| SHA-256 | 8d13f085128f27c446664b1847efebd02f5383743fd053a9ae6dd05a4c49327d |
