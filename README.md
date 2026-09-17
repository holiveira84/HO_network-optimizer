# NetworkOptimizer — Casa Oliveira

Deploy próprio de [Ozark-Connect/NetworkOptimizer](https://github.com/Ozark-Connect/NetworkOptimizer),
uma ferramenta de auditoria de segurança e otimização para redes UniFi: 83 verificações de
segurança, otimização de canais WiFi, monitorização de WAN/ISP, testes de velocidade e
estatísticas de modem/ONT/SFP.

Instalado em **2026-09-16** como teste (a pedido do utilizador, "para explorar o painel antes de
decidir avançar") — sem qualquer alteração de rede feita remotamente; a ligação ao controller
UniFi foi feita pelo próprio utilizador, com uma conta local dedicada criada por ele na consola
UniFi.

## Onde corre

LXC dedicada no Proxmox (`pve`, 192.168.2.112), **VMID 118**, Debian 12, Docker dentro da LXC
(`nesting=1,keyctl=1`, unprivileged), disco **inteiramente no `tank1`** (não no SSD/`local-lvm` —
política da casa: o SSD é só para o sistema Proxmox).

- **Acesso:** http://192.168.2.224:8042 (LAN apenas)
- **Imagem:** `ghcr.io/ozark-connect/network-optimizer:latest`
- **Autenticação:** password fixa via variável de ambiente `APP_PASSWORD` (não o fluxo de password
  auto-gerada da instalação oficial)
- **Dados persistentes:** `/opt/networkoptimizer/{data,logs}` no host da LXC

## Porquê a instalação manual (não o script oficial)

O instalador oficial do projeto — tanto o `scripts/proxmox/install.sh` do próprio repo como o
script do [community-scripts.org](https://community-scripts.org/scripts/networkoptimizer) (o
"Proxmox VE Community Scripts", sucessor do tteck) — usa diálogos interativos (`whiptail`), que
não funcionam por SSH não-interativo. Em vez disso, a LXC foi criada manualmente com as mesmas
specs recomendadas pelo instalador oficial (2 CPU, 4096MB RAM, 12GB disco), Docker instalado
diretamente, e o container lançado via `docker run` a partir do `docker-compose.prod.yml` do
repositório (só o serviço principal `network-optimizer`, sem o sidecar de speedtest por agora).

```bash
docker run -d --name network-optimizer \
  -e TZ=Europe/Lisbon \
  -e HOST_IP=192.168.2.224 \
  -e Iperf3Server__Enabled=true \
  -p 8042:8042 \
  -p 5201:5201 -p 5201:5201/udp \
  -v /opt/networkoptimizer/data:/app/data \
  -v /opt/networkoptimizer/logs:/app/logs \
  --restart=unless-stopped \
  ghcr.io/ozark-connect/network-optimizer:latest
```

(`APP_PASSWORD` só é necessária na primeira criação — o admin já fica gravado em `/app/data`; recriações
seguintes do container não precisam de a repassar. Comando acima já inclui `Iperf3Server__Enabled` e a
porta 5201, ver secção "Testes de velocidade LAN" abaixo.)

## Ligação ao UniFi

Feita pelo utilizador diretamente no painel (Settings → UniFi Connection), com uma conta **local**
dedicada criada na consola UniFi (não SSO — o próprio guia recomenda isto para permitir níveis de
acesso restritos: `Network View Only` + `Protect View Only` em vez de Super Admin, já que a maior
parte das funcionalidades são só de leitura). Confirmado a funcionar via logs do container:

```
Successfully authenticated with UniFi controller
Retrieved system info - Controller: UDM Pro HOLIVEIRA v10.6.101
```

## SSH da gateway (ligado em 2026-09-17)

O NetworkOptimizer usa SSH à gateway só para funcionalidades que dependem de correr comandos nela
(testes iperf3, Adaptive SQM para bufferbloat, WAN speed test) — não é necessário para auditoria,
WiFi ou monitorização (essas usam só a API do controller). Inicialmente deixado por ligar; ativado
depois a pedido explícito do utilizador.

**Chave gerida:** usada a funcionalidade "Managed SSH Key" do próprio NetworkOptimizer (Settings →
Connection) — gera um par de chaves Ed25519 dentro do próprio container; a chave privada nunca sai
de lá, só a pública precisa de ser instalada no destino.

**Causa raiz do "Connection refused" inicial:** o UDM Pro tem **dois toggles de SSH totalmente
independentes**:
- `mgmt.x_ssh_enabled` (API/app Network) — controla o SSH dos *dispositivos geridos*
  (switches/APs), já estava ativo, usado pelo utilizador `home-assistant` existente.
- **SSH da própria consola UniFi OS** — em `Settings → Control Plane → Console → SSH`, secção
  separada da app Network, com o próprio utilizador `root` e password dedicada. Estava
  **desligado**, e era isso que bloqueava a porta 22 por completo (testado com `nc`/`/dev/tcp` a
  partir do host Proxmox e da própria LXC — "Connection refused" nos dois, confirmando que não era
  problema de firewall/zona, mas o daemon SSH da consola mesmo por ligar).

**Correção:** ativado o SSH da consola via a UI do UniFi (não há endpoint de API direto para isto),
password nova definida (guardada no vault, `unifi.ssh_console_root_password`). No NetworkOptimizer,
o "Gateway SSH" foi configurado com utilizador `root` + essa password (não a chave gerida, que é só
para dispositivos) — é a combinação que o próprio painel do NetworkOptimizer indica para gateways
UDM/UCG/UDR. "Test SSH Connection" confirmou sucesso.

## Testes de velocidade LAN (iperf3 + OpenSpeedTest, 2026-09-17)

Dois testes "de qualquer dispositivo" ativados, conforme a própria página `/speedtest` do
NetworkOptimizer descreve:

- **Browser Speed Test (OpenSpeedTest):** container sidecar dedicado
  `ghcr.io/ozark-connect/speedtest:latest` (fork customizado do OpenSpeedTest que envia os
  resultados automaticamente para o NetworkOptimizer), a correr em `http://192.168.2.224:3005`.
  Qualquer dispositivo na LAN abre a página e clica "Start" — o resultado aparece no "Test History"
  do painel principal, identificado pelo IP de origem e associado ao cliente UniFi correspondente.
  ```bash
  docker run -d --name network-optimizer-speedtest \
    -e TZ=Europe/Lisbon -e HOST_IP=192.168.2.224 -e OPENSPEEDTEST_PORT=3005 \
    -p 3005:3000 --restart=unless-stopped \
    --sysctl net.ipv4.tcp_rmem="4096 131072 33554432" \
    --sysctl net.ipv4.tcp_wmem="4096 65536 33554432" \
    --sysctl net.ipv4.tcp_mtu_probing=1 \
    ghcr.io/ozark-connect/speedtest:latest
  ```
  Testado com sucesso via browser: **947.3 Mbps download / 949.3 Mbps upload / 1ms ping / 0.1ms jitter**.

- **iperf3 to Server:** não é um container à parte — o próprio binário `network-optimizer` tem um
  servidor iperf3 embutido, ativado com `Iperf3Server__Enabled=true` e a porta 5201 (TCP+UDP)
  publicada (ver `docker run` acima). Testado a partir deste PC Windows (`winget install
  ar51an.iPerf3`, `iperf3 -c 192.168.2.224 -p 5201`): **~937 Mbits/sec**, resultado também gravado
  no "Test History" (dispositivo `OLIVEIRA`, 935.0 Mbps, 8s, associado ao caminho de switches real:
  Escritório → US 8 150W → US 24 → Garagem).

## InfluxDB (monitorização de séries temporais)

O NetworkOptimizer pedia uma instância própria de InfluxDB para guardar métricas (contadores de
porta, latência, níveis óticos SFP). Em vez de criar uma instância nova, foi reaproveitado o
**InfluxDB já existente** (LXC 114, usado para os dados solares FoxESS/Home Assistant — ver
[`HO_proxmox-pve`](https://github.com/holiveira84/HO_proxmox-pve)), org `casa-oliveira`.

O próprio NetworkOptimizer criou os seus buckets dedicados durante o setup (fluxo: dar-lhe
temporariamente o token "all-access" da organização, que ele usa só nesse momento para criar os
buckets e gerar um token de privilégio mínimo próprio, descartando o all-access logo a seguir):

- `network_monitoring` (retenção 90 dias)
- `network_monitoring_longterm` (retenção 365 dias)

(Uma tentativa inicial de criar manualmente um único bucket `networkoptimizer` com um token
pré-scoped foi feita antes de se perceber que o fluxo de setup da própria app precisa do token
all-access para se auto-provisionar — esse bucket/token manuais foram apagados depois, sem uso.)

## Modem/ONT

Não foi encontrado nenhum dispositivo identificável como modem/ONT na lista de clientes do UniFi —
expectável, já que o ONT fica tipicamente a montante do UDM (entre a operadora e o router), fora
da rede LAN gerida pelo UniFi. Secção "Monitoring Interfaces" do NetworkOptimizer deixada vazia por
agora; as estatísticas óticas/SFP diretas do modem ficam por adicionar se/quando o utilizador
confirmar o IP de gestão físico do aparelho.

## Adaptive SQM (configurado, deployment por fazer)

Pré-requisito "Smart Queues Required" resolvido: ativado em `Settings → Internet → MEO → Advanced →
Smart Queues` no UniFi, com Downrate/Uprate 500/100 Mbps (velocidade real da linha). **Trade-off
conhecido e aceite:** com o SQM ativo, o throughput caiu de ~516/120 Mbps para ~380/84 Mbps — o SQM
clássico da UDM Pro corre em software (fq_codel/cake, CPU-bound), limitação de hardware conhecida em
linhas rápidas. Decisão do utilizador: manter ativo, aceitar a perda em troca de latência mais
estável sob carga.

O painel `/sqm` do NetworkOptimizer foi preenchido (MEO, DOCSIS Cable, 500/100 Mbps nominal, "Enable
Adaptive SQM") mas **o deploy não foi feito** — esse painel só grava a configuração no momento de
"Deploy SQM Scripts" (não há guardar rascunho), que instala scripts + cron jobs no próprio gateway
para shaping ativo em tempo real. Fica pendente de decisão do utilizador.

## Estado (2026-09-17)

- ✅ LXC criada, Docker instalado, container saudável
- ✅ Ligado ao UniFi (UDM Pro HOLIVEIRA v10.6.101) com conta local dedicada
- ✅ InfluxDB configurado (buckets próprios criados pela app)
- ✅ Auditoria de segurança inicial corrida (score 18/100, 130 achados) — levou a corrigir o
  isolamento real das VLANs R1-R7 e ativar DNS-over-HTTPS no UniFi (detalhe em
  [`HO_proxmox-pve`](https://github.com/holiveira84/HO_proxmox-pve) / vault, secção `unifi`)
- ✅ SSH da gateway ligado — iperf3/Adaptive SQM/WAN speed test desbloqueados
- ✅ Testes de velocidade LAN (browser OpenSpeedTest + iperf3) ativados e testados com sucesso
- ✅ Smart Queues (UniFi) ativado — pré-requisito do Adaptive SQM
- ⏳ Adaptive SQM configurado na UI mas sem deploy (ver secção acima)
- ⏳ Modem/ONT não configurado (IP de gestão não encontrado)
- ⏳ IP ainda em DHCP (192.168.2.224) — decisão de fixar/avançar a sério fica pendente do
  utilizador após explorar o painel
