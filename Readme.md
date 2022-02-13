# Jitsi 

Jitsi é uma plataforma de video conferencia com diversos recursos que poderá ser implementado como uma robusta opção de conferencia corporativa.

## Proposito

Está documentação é um guia que poderá ser usado no curso, mas tudos isso poderá ser configurado de forma modular.

## Instalação

....

# Jibri

Usado para fazer gravação e livestream, ideal este cara está apartado do cluster.

Este componente pode ser instalado e configurado no próprio servidor Jitsi ou em um servidor apartado.

## Kernel Extra

Para instalação do kernel na versão que estamos usando do Ubuntu 18:

```md
sudo apt install linux-image-extra-virtual
```

Carregando o kernel no grub: 

```md
$ sudo grep 'menuentry ' /boot/grub/grub.cfg | cut -f 2 -d "'" | nl -v 0
     0	Ubuntu, with Linux 4.15.0-167-generic
     1	Ubuntu, with Linux 5.4.0-1069-azure
     2	Ubuntu, with Linux 5.4.0-1069-azure (recovery mode)
     3	Ubuntu, with Linux 5.4.0-1068-azure
     4	Ubuntu, with Linux 5.4.0-1068-azure (recovery mode)
     5	Ubuntu, with Linux 5.4.0-1067-azure
     6	Ubuntu, with Linux 5.4.0-1067-azure (recovery mode)
     7	Ubuntu, with Linux 5.3.0-1034-azure
     8	Ubuntu, with Linux 5.3.0-1034-azure (recovery mode)
     9	Ubuntu, with Linux 4.15.0-167-generic
    10	Ubuntu, with Linux 4.15.0-167-generic (recovery mode)
```

Selecione o kernel que deseja carregar:

```md 
$ sudo grub-set-default 9
$ sudo update-grub2
$ sudo grub-reboot 9
$ sudo reboot
```

Ou da maneira que fiz que resolver de imediato:

```md
$ sudo vim /boot/grub/grub.cfg
... 

menuentry 'Ubuntu, with Linux 4.15.0-167-generic' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-4.15.0-167-generic-advanced-f3b1f9ec-6c9d-4e76-adc4-17b250df8e24' {
        recordfail
        load_video
        gfxmode $linux_gfx_mode
        insmod gzio
        if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
        insmod part_gpt
        insmod ext2
        set root='hd0,gpt1'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-bios=hd0,gpt1 --hint-efi=hd0,gpt1 --hint-baremetal=ahci0,gpt1  f3b1f9ec-6c9d-4e76-adc4-17b250df8e24
        else
          search --no-floppy --fs-uuid --set=root f3b1f9ec-6c9d-4e76-adc4-17b250df8e24
        fi
        echo    'Loading Linux 4.15.0-167-generic ...'
        linux   /boot/vmlinuz-4.15.0-167-generic root=UUID=f3b1f9ec-6c9d-4e76-adc4-17b250df8e24 ro  console=tty1 console=ttyS0 earlyprintk=ttyS0
        echo    'Loading initial ramdisk ...'
        initrd  /boot/initrd.img-4.15.0-167-generic
}

...
```

> Na sequencia já dei um reboot no host.


A importancia de instalar este kernel é carregar o modulo `snd_loop` para usabilidade de driver de som.

Para que o mesmo seja carregado, terá que fazer alguns comandos:

```md 
$ sudo modprobe snd-aloop
$ sudo echo "snd-aloop" >> /etc/modules
```

Este é o final da parte de kernel e módulo.

## Chrome stable

Neste servidor será instalado o google chrome stable e a parte gráfica para funcionalidade da gravação, importando a chave GPG do repositório, configurando repositório e instalando:

```md
$ curl https://dl-ssl.google.com/linux/linux_signing_key.pub | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/google-chrome-keyring.gpg'

$ echo "deb [arch=amd64 signed-by=/usr/share/keyrings/google-keyring.gpg] http://dl.google.com/linux/chrome/deb/ stable main" | sudo tee /etc/apt/sources.list.d/google-chrome.list

$ sudo apt-get -y update
$ sudo apt-get -y install google-chrome-stable
```

Ou instalando desta forma, caso o exemplo acima tenha dificuldade:

```md 
$ wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
$ dpkg -i google-chrome-stable_current_amd64.deb
$ sudo apt install -f
```


## Chrome driver

Para instalação dele é bem simples, crie a estrutura de diretório:

```md
$ mkdir -p /etc/opt/chrome/policies/managed
$ echo '{ "CommandLineFlagSecurityWarningsEnabled": false }' >>/etc/opt/chrome/policies/managed/managed_policies.json
$ CHROME_DRIVER_VERSION=`curl -sS chromedriver.storage.googleapis.com/LATEST_RELEASE`
$ wget -N http://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip -P ~/
```

E é preciso fazer a instalação do unzip no ubuntu e descompactar o arquivo e jogar no diretorio de binarios:

```md
$ sudo apt install unzip
$ sudo unzip ~/chromedriver_linux64.zip -d ~/
$ sudo mv -f ~/chromedriver /usr/local/bin/chromedriver
$ sudo chown root:root /usr/local/bin/chromedriver
$ sudo chmod 0755 /usr/local/bin/chromedriver

```

### Outras Dependencias

Para instalação de outras dependencias neste servidor:

```md
$ sudo apt-get install default-jre-headless ffmpeg curl alsa-utils icewm xdotool xserver-xorg-video-dummy ruby-hocon
```

### Instalando o Jibri

Para instalação vamos configurar o repositorio e logo após o pacote jibri:

```md
$ curl https://download.jitsi.org/jitsi-key.gpg.key | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'
$ cho 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/' | sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install jibri
```

Adicionando o usuario jibri nos grupos, lembrando que é melhor executar, mesmo vc olhando que já foi adicionado:

```md
$ sudo usermod -aG adm,audio,video,plugdev jibri
```

Agora vamos seguir com os arquivos de configuração:

### Prosody

Criando os usuários:

```md
$ prosodyctl register jibri auth.pocconferencia.eurekadigital.app jibriauthpass
$ prosodyctl register recorder recorder.pocconferencia.eurekadigital.app  jibrirecorderpass
```

### Jicofo

Para configuração do Jicofo:

```md

```

### Referenica Sites

https://github.com/jitsi/jibri

https://community.jitsi.org/t/tutorial-how-to-install-the-new-jibri/88861

### Referenica Video