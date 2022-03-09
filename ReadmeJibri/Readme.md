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
$ sudo dpkg -i google-chrome-stable_current_amd64.deb
$ sudo apt install -f
```

## Chrome driver

Para instalação dele é bem simples, crie a estrutura de diretório:

```md
$ sudo mkdir -p /etc/opt/chrome/policies/managed
$ sudo echo '{ "CommandLineFlagSecurityWarningsEnabled": false }' >>/etc/opt/chrome/policies/managed/managed_policies.json
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
$ sudo curl https://download.jitsi.org/jitsi-key.gpg.key | sudo sh -c 'gpg --dearmor > /usr/share/keyrings/jitsi-keyring.gpg'
$ sudo echo 'deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/' | sudo tee /etc/apt/sources.list.d/jitsi-stable.list > /dev/null
$ sudo apt-get update
$ sudo apt-get install jibri
```

Liberando a porta de serviço:

```md
$ sudo ufw allow 5222/tcp
```

Adicionando o usuario jibri nos grupos, lembrando que é melhor executar, mesmo vc olhando que já foi adicionado:

```md
$ sudo usermod -aG adm,audio,video,plugdev jibri
```

Preparando o diretório que estará os videos gravados:

```md
$ mkdir /srv/recordings
$ chown jibri:jibri /srv/recordings
```

Agora vamos seguir com os arquivos de configuração:

```md
$ sudo vim /etc/jitsi/jibri/jibri.conf

jibri {
  // A unique identifier for this Jibri
  // TODO: eventually this will be required with no default
  id = ""
  // Whether or not Jibri should return to idle state after handling
  // (successfully or unsuccessfully) a request.  A value of 'true'
  // here means that a Jibri will NOT return back to the IDLE state
  // and will need to be restarted in order to be used again.
  single-use-mode = false
  api {
    http {
      external-api-port = 2222
      internal-api-port = 3333
    }
    xmpp {
      // See example_xmpp_envs.conf for an example of what is expected here
      environments = [
	      {
                name = "prod environment"
                xmpp-server-hosts = ["nosso.dominio.app"]
                xmpp-domain = "nosso.dominio.app"

                control-muc {
                    domain = "internal.auth.nosso.dominio.app"
                    room-name = "JibriBrewery"
                    nickname = "jibri-nickname"
                }

                control-login {
                    domain = "auth.nosso.dominio.app"
                    username = "jibri"
                    password = "jibriauthpass"
                }

                call-login {
                    domain = "recorder.nosso.dominio.app"
                    username = "recorder"
                    password = "jibrirecorderpass"
                }

                strip-from-room-domain = "conference."
                usage-timeout = 0
                trust-all-xmpp-certs = true
            }]
    }
  }
  recording {
    recordings-directory = "/srv/recordings"
    # TODO: make this an optional param and remove the default
    finalize-script = "/path/to/finalize_recording.sh"
  }
  streaming {
    // A list of regex patterns for allowed RTMP URLs.  The RTMP URL used
    // when starting a stream must match at least one of the patterns in
    // this list.
    rtmp-allow-list = [
      // By default, all services are allowed
      ".*"
    ]
  }
  chrome {
    // The flags which will be passed to chromium when launching
    flags = [
      "--use-fake-ui-for-media-stream",
      "--start-maximized",
      "--kiosk",
      "--enabled",
      "--disable-infobars",
      "--autoplay-policy=no-user-gesture-required"
    ]
  }
  stats {
    enable-stats-d = true
  }
  webhook {
    // A list of subscribers interested in receiving webhook events
    subscribers = []
  }
  jwt-info {
    // The path to a .pem file which will be used to sign JWT tokens used in webhook
    // requests.  If not set, no JWT will be added to webhook requests.
    # signing-key-path = "/path/to/key.pem"

    // The kid to use as part of the JWT
    # kid = "key-id"

    // The issuer of the JWT
    # issuer = "issuer"

    // The audience of the JWT
    # audience = "audience"

    // The TTL of each generated JWT.  Can't be less than 10 minutes.
    # ttl = 1 hour
  }
  call-status-checks {
    // If all clients have their audio and video muted and if Jibri does not
    // detect any data stream (audio or video) comming in, it will stop
    // recording after NO_MEDIA_TIMEOUT expires.
    no-media-timeout = 30 seconds

    // If all clients have their audio and video muted, Jibri consideres this
    // as an empty call and stops the recording after ALL_MUTED_TIMEOUT expires.
    all-muted-timeout = 10 minutes

    // When detecting if a call is empty, Jibri takes into consideration for how
    // long the call has been empty already. If it has been empty for more than
    // DEFAULT_CALL_EMPTY_TIMEOUT, it will consider it empty and stop the recording.
    default-call-empty-timeout = 30 seconds
  }
}

```

### Prosody

Criando os usuários:

```md
$ prosodyctl register jibri auth.nosso.dominio.app jibriauthpass
$ prosodyctl register recorder recorder.nosso.dominio.app  jibrirecorderpass
```

Configurando e criando uma MUC interna e o VirtualHost, Componentes, no final do arquivo:

```md
$ sudo vim /etc/prosody/conf.d/nosso.dominio.app.cfg.lua

-- Jibri

Component "internal.auth.nosso.dominio.app" "muc"
    modules_enabled = {
      "ping";
    }
    -- storage should be "none" for prosody 0.10 and "memory" for prosody 0.11
    storage = "none"
    muc_room_cache_size = 1000

VirtualHost "recorder.nosso.dominio.app"
  modules_enabled = {
    "ping";
  }
  authentication = "internal_plain"    

```

### Jicofo

Para configuração do Jicofo é preciso adicionar o tempo que ele irá esperar, duas ultimas linhas:

```md
$ vim /etc/jitsi/jicofo/sip-communicator.properties
org.jitsi.jicofo.BRIDGE_MUC=JvbBrewery@internal.auth.nosso.dominio.app
org.jitsi.jicofo.jibri.BREWERY=JibriBrewery@internal.auth.nosso.dominio.app
org.jitsi.jicofo.jibri.PENDING_TIMEOUT=90
```

### Jitsi Meet

Nesta reta final será preciso apenas habilitar o frontend para ter a opção de recording e liveness habilitada:

```md
$ sudo vim /etc/jitsi/meet/nosso.dominio.app

fileRecordingsEnabled: true, // If you want to enable file recording
liveStreamingEnabled: true, // If you want to enable live streaming
hiddenDomain: 'recorder.yourdomain.com', // Esta opcao é adicionada
```

### Referenica Sites

https://github.com/jitsi/jibri

https://community.jitsi.org/t/tutorial-how-to-install-the-new-jibri/88861

### Referenica Video
