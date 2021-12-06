---
layout: post
title: Chrome Remote Terminal - Chrome Remote Desktop mínimo no Linux
tags: linux chrome remote desktop
---

O [Chrome Remote Desktop](https://remotedesktop.google.com/), ou "Área de trabalho remota do Google Chrome", é uma excelente ferramenta de acesso remoto disponível para Windows, Linux e MacOS.

Mas existem ocasiões em que você não necessita de um ambiente desktop completo para acessar remotamente. Que tal tentar configurar um ambiente Linux com o mínimo consumo de recursos possível para ter um "Chrome Remote Terminal"?

Como o *Chromoting* para Linux é distribuído num pacote **.deb**, idealmente você deve ter uma máquina rodando o [Debian](https://www.debian.org/). Como sugestão, instale um sistema bem mínimo apenas com o básico do necessário a partir do [Debian netinst CD image](https://www.debian.org/CD/netinst/index.en.html).

Depois que o sistema básico estiver instalado, é hora de instalar o Chrome Remote Desktop, um gerenciador de janelas e um emulador de terminal. Para os dois últimos, as opções que julgo serem as mais apropriadas (e mínimas) são o [Matchbox](https://www.yoctoproject.org/software-item/matchbox/) e o [st](https://st.suckless.org/).

```bash
# Baixar e instalar o Chrome Remote Desktop
wget https://dl.google.com/linux/direct/chrome-remote-desktop_current_amd64.deb
sudo apt-get install --assume-yes ./chrome-remote-desktop_current_amd64.deb
# Instalar o Matchbox e o st
sudo apt install matchbox-window-manager stterm
# Configurar a sessão do Chromoting
sudo bash -c 'echo "exec matchbox-window-manager -use_titlebar no & st" > /etc/chrome-remote-desktop-session'
```

Por fim, vincule o sistema com sua conta do Google seguindo as instruções na página [https://remotedesktop.google.com/headless](https://remotedesktop.google.com/headless)

---

Leitura complementar: [Setting up Chrome Remote Desktop for Linux on Compute Engine](https://cloud.google.com/architecture/chrome-desktop-remote-on-compute-engine)