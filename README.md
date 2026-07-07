![Platform](https://img.shields.io/badge/platform-Raspberry%20Pi%20Zero%202%20W-c51a4a)
![Stream](https://img.shields.io/badge/stream-MJPEG-orange)
![License](https://img.shields.io/badge/license-MIT-green)

# Câmera Zenital — IARA++ Lab

Sistema de streaming de vídeo instalado no teto do laboratório IARA++ (UFABC), transmitindo por Wi-Fi via Raspberry Pi Zero 2 W + [ustreamer](https://github.com/pikvm/ustreamer). Qualquer projeto do laboratório pode consumir esse stream — não é exclusivo de um experimento específico.

## Acesso rápido

```
http://10.0.0.189:8080/
```

> ⚠️ Preencher o IP atual da Pi na rede do lab. Se a rede usa DHCP, o IP pode mudar após reinicializações — ver seção [Encontrar o IP](#encontrar-o-ip-quando-mudar) abaixo.

Abra esse endereço em qualquer navegador conectado ao Wi-Fi do laboratório para ver o vídeo ao vivo. Para consumo programático (Python, OpenCV, ROS, etc.), o endpoint de stream MJPEG é `http://10.0.0.189:8080/stream`.

## Visão geral

Uma câmera USB fixada no teto do laboratório, conectada a uma Raspberry Pi Zero 2 W, transmite vídeo MJPEG pela rede Wi-Fi. A Pi funciona como um appliance: liga, conecta automaticamente na rede do lab, e sobe o stream sozinha — sem precisar de monitor, teclado ou intervenção manual.

```
[Câmera USB, teto] --USB--> [Pi Zero 2 W] --ustreamer/MJPEG--> [Wi-Fi do lab] --> [qualquer consumidor na rede]
```

### Casos de uso possíveis

- Visão zenital para tracking de robôs (ex: detecção de marcadores ArUco para ground truth de trajetória)
- Monitoramento remoto da bancada/setup de testes
- Gravação de experimentos para documentação
- Qualquer projeto que precise de uma câmera fixa compartilhada, sem precisar montar hardware próprio

## Estrutura do repositório

```
ceiling-camera-lab/
├── README.md
├── LICENSE
├── .gitignore
└── systemd/
    └── ustreamer.service
```

## Encontrar o IP (quando mudar)

Se a Pi reiniciar ou o DHCP renovar o IP, encontre o endereço atual:

- **App Fing** (celular, conectado no Wi-Fi do lab): lista dispositivos por fabricante — a Pi aparece como "vitor"
- **arp-scan** (notebook, mesma rede):
  ```bash
  sudo apt install arp-scan
  sudo arp-scan --localnet
  ```
- **Hostname configurado**, se `.local` (mDNS) funcionar na rede do lab:
  ```
  http://<hostname>.local:8080
  ```

Recomenda-se, se possível, configurar um **IP fixo (reserva DHCP)** no roteador do lab para esse dispositivo, evitando que o endereço mude e quebre o acesso de quem depende do stream.

## Hardware

| Item | Observação |
|---|---|
| Raspberry Pi Zero 2 W | Porta de dados é micro-USB, não USB-A |
| Câmera USB | Qualquer câmera UVC compatível com MJPEG (testado com XWF-1080P, até 1920x1080@30fps) |
| Adaptador OTG micro-USB → USB-A | Necessário para conectar a câmera na porta de **dados** (a do meio, marcada "USB") |
| Cartão microSD | 8GB+ |

## Setup do zero (para reconfiguração ou nova unidade)

### 1. Gravação do cartão SD

Usando o **Raspberry Pi Imager**, escolher **Raspberry Pi OS Lite (32-bit)** — sem ambiente gráfico, já que o único serviço rodado é o ustreamer.

Na engrenagem ⚙️ (Edit settings):
- Hostname
- Habilitar SSH (senha ou chave)
- Wi-Fi: SSID + senha + país **BR**
- Usuário e senha

> A Pi Zero 2 W só tem rádio **2,4 GHz** — conectar sempre na banda de 2,4 GHz da rede do lab.

### 2. Primeiro acesso via SSH

```bash
ssh usuario@hostname.local
```

Se `.local` não resolver, usar as mesmas opções da seção [Encontrar o IP](#encontrar-o-ip-quando-mudar).

### 3. Adicionar a rede do laboratório remotamente

Possível configurar o perfil Wi-Fi do lab sem estar fisicamente lá — a Pi conecta sozinha quando chegar perto:

```bash
sudo nmcli connection add type wifi con-name "lab" ifname wlan0 \
  ssid "NOME_DA_REDE_LAB" \
  wifi-sec.key-mgmt wpa-psk \
  wifi-sec.psk "SENHA_DO_LAB"
```

Priorizar essa rede quando ambas estiverem no alcance:

```bash
sudo nmcli connection modify "lab" connection.autoconnect-priority 10
```

> Cobre redes **WPA-PSK simples**. Redes corporativas (WPA2-Enterprise / eduroam) exigem parâmetros adicionais (`802-1x`).

### 4. Instalação do ustreamer

```bash
sudo apt update && sudo apt install -y git build-essential libevent-dev libjpeg-dev libbsd-dev v4l-utils
git clone https://github.com/pikvm/ustreamer
cd ustreamer
make -j4
```

### 5. Verificação da câmera

```bash
v4l2-ctl --list-devices
```

A câmera deve aparecer associada a um `/dev/videoX` (geralmente `/dev/video0`). Para ver resoluções/FPS suportados:

```bash
v4l2-ctl --device=/dev/video0 --list-formats-ext
```

> Se aparecer `Device doesn't support setting of HW encoding quality parameters`: é porque a câmera já comprime MJPEG em hardware — o parâmetro `--quality` do ustreamer é ignorado nesse caso, sem prejuízo.

### 6. Teste manual

```bash
cd ~/ustreamer
./ustreamer --device=/dev/video0 --host=0.0.0.0 --port=8080 --resolution=640x480 --desired-fps=30 --format=MJPEG
```

Acessar `http://IP_DA_PI:8080` de qualquer dispositivo na rede.

**Resolução/FPS atual: 640x480 @ 30fps** — priorizando FPS alto (amostragem temporal densa) sobre resolução, mantendo leveza para a CPU/rede da Zero. Ajustável conforme a necessidade de cada projeto (ver seção seguinte).

### 7. Serviço systemd (autostart)

```bash
sudo cp systemd/ustreamer.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable ustreamer.service
sudo systemctl start ustreamer.service
sudo systemctl status ustreamer.service
```

Deve mostrar `active (running)` e `enabled`.

### Alterar resolução/FPS

Os parâmetros ficam na linha `ExecStart` do arquivo de serviço:

```bash
sudo nano /etc/systemd/system/ustreamer.service
# editar ExecStart
sudo systemctl daemon-reload
sudo systemctl restart ustreamer.service
```

> Como o stream é compartilhado entre projetos, combine com outros usuários do lab antes de mudar a configuração padrão — um ajuste pode afetar quem já depende do stream atual.

## Resolução de problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| Câmera não aparece em `v4l2-ctl --list-devices` | Adaptador OTG na porta errada (energia em vez de dados) | Usar a porta micro-USB do meio ("USB"), não a "PWR" |
| Não acha o IP da Pi na rede do lab | mDNS (`.local`) bloqueado pela rede | Usar Fing, `arp-scan`, ou painel do roteador |
| Stream trava/atrasa em movimento | Resolução alta demais para a banda de 2,4 GHz | Reduzir resolução (ex: 640x480) e manter FPS alto |
| `ERROR: Device doesn't support setting of HW encoding quality parameters` | Câmera já comprime MJPEG em hardware | Comportamento esperado, não é falha — `--quality` é ignorado |

## Próximos passos

- [ ] **Validar o delay (latência) do sistema.** Ainda não foi medido o atraso entre o movimento real e o frame recebido pelo cliente — importante para qualquer aplicação sensível a tempo (ex: sincronização com dados de odometria). Setup ainda não deve ser considerado 100% validado até essa medição.
- [ ] Configurar IP fixo (reserva DHCP) para o dispositivo na rede do lab.
- [ ] Avaliar necessidade de autenticação/restrição de acesso ao stream, se o lab tiver rede compartilhada com outros grupos.

## Créditos

Este projeto usa o [**µStreamer (ustreamer)**](https://github.com/pikvm/ustreamer), desenvolvido por Maxim Devaev como parte do projeto [Pi-KVM](https://pikvm.org/), distribuído sob licença **GPL-3.0-or-later**. É utilizado aqui como dependência externa (binário compilado a partir do código-fonte oficial), sem modificações — este repositório documenta apenas a configuração e o uso da ferramenta como infraestrutura compartilhada do laboratório.

Consulte o repositório original para documentação completa e opções avançadas (encoding via hardware M2M, suporte a Janus/WebRTC, etc.).

## Licença

O conteúdo deste repositório (documentação e arquivo de configuração systemd) está sob licença MIT — veja [LICENSE](LICENSE). O uso do ustreamer em si permanece sujeito à licença GPL-3.0-or-later do projeto original.
