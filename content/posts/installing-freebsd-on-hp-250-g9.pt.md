+++ 
date = 2024-10-28T18:49:00+01:00
title = "Instalando o FreeBSD em um HP 250 G9"
categories = ["Operating Systems"]
+++

Comprei um notebook Hewlett-Packard 250 G9 de 15,6 polegadas com um Intel Core i3-1215U de 12ª geração, 8GB de RAM e um SSD de 500GB. Não é um hardware impressionante, admito, mas eu não vou rodar muita coisa localmente mesmo. Então, vamos direto ao ponto.

Instalei o FreeBSD 14.1. Na verdade, não é tão assustador quanto parece. O processo de instalação é simples e fácil de entender. A parte mais complexa seria o particionamento, mas, no meu caso, usei o disco inteiro — sem necessidade de dual boot — e deixei a ferramenta fazer o trabalho sozinha.

Não vou adicionar screenshots de cada passo, já que pessoas muito mais experientes já cobriram isso a fundo. Dê uma olhada [neste vídeo](https://www.youtube.com/watch?v=bQKaNbarQKI), do [RoboNuggie](https://www.youtube.com/@RoboNuggie). Cerca de 70% de tudo o que eu precisei fazer está explicado nele.

Vou listar abaixo o que não foi coberto no vídeo, como reprodução de mídia protegida, vídeo, áudio e configuração do WiFi.

## Vídeo
Este notebook tem a Alder Lake-UP3 GT1 [UHD Graphics], que requer o `drm-61-kmod`.
```
$ cd /usr/ports/graphics/drm-61-kmod
$ sudo make install clean
$ sudo sysrc kld_list+=i915kms
```

## Gerenciador de janelas
Estou usando o Mate; para isso, simplesmente segui o [FreeBSD Handbook](https://docs.freebsd.org/en/books/handbook/desktop/).

## Chromium e CDM
Encontrei alguns tutoriais na internet, mas aqui vou compartilhar passos mais simples para habilitar a reprodução de mídia protegida — Netflix, Spotify etc. — no Chromium:
- Instale o Chromium e o [Foreign CDM](https://www.freshports.org/www/foreign-cdm):
```
$ sudo pkg install foreign-cdm chromium
$ sudo sysrc linux_enable="YES"
$ sudo service linux start
```
- Baixe os ports e o código-fonte do FreeBSD para compilar o `www/linux-widevine-cdm`.
```
$ sudo pkg update
$ sudo pkg install portsnap git
$ sudo portsnap fetch
$ sudo portsnap extract
$ sudo git clone -b releng/14.1 https://git.freebsd.org/src.git /usr/src
```
- Instale o [Linux Widevine CDM](https://www.freshports.org/www/linux-widevine-cdm/?branch=2023Q4)
```
$ cd /usr/ports/www/linux-widevine-cdm
$ sudo make install clean
```
- Abra `chrome://flags` no Chromium e habilite `Cdm Storage Database` e `Cdm Storage Database Migration`:
- Feche o Chromium e abra de novo.
- Você pode testar se o DRM está habilitado em https://bitmovin.com/demos/drm.

## Impressora
Tenho uma HP OfficeJet e, pela experiência passada com Linux, eu já conhecia o [HPLIP](https://developers.hp.com/hp-linux-imaging-and-printing). Felizmente, ele também estava disponível como binário, bastando instalar com `pkg install hplip`.

## Áudio
A configuração do áudio também é simples. Precisei carregar um driver e configurar a troca automática dos alto-falantes para o fone de ouvido, e vice-versa, sempre que um dispositivo era (des)conectado na entrada de áudio. Segui [este tutorial](https://freebsdfoundation.org/resource/audio-on-freebsd-quick-guide/).

## Wifi
Esta foi a parte mais difícil da configuração. Minha primeira tentativa foi carregar o driver compatível com a minha placa. Aparentemente, o chipset era suportado, e funcionou no começo. Mas, assim que comecei a assistir vídeos, ficou instável. Tentei um micro adaptador WiFi, mas era bem lento. Depois de muita leitura em fóruns e algumas dores de cabeça, descobri o [projeto Wifibox](https://github.com/pgj/freebsd-wifibox).

É uma ideia engenhosa: rodar os drivers de WiFi dentro de uma VM Linux convidada e fazer uma ponte da conexão de volta para a máquina host. O consumo de recursos é bem pequeno, e consigo usar todo o potencial do meu WiFi. Segui [este tutorial](https://xyinn.org/md/freebsd/wifibox).

## Componentes não testados
Não testei o Bluetooth nem a saída de áudio via HDMI.

## Plataformas testadas:
- Zoom
- Webex
- GeForce Now
- Netflix
- Amazon Primevideo
- Spotify

Bom, é isso: com alguns passos manuais, tenho um sistema operacional que atende às minhas necessidades.

**See you | Bis geschwënn | Até mais | À bientôt | Ciao**
