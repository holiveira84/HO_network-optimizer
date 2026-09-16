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
  -e APP_PASSWORD=<password> \
  -p 8042:8042 \
  -v /opt/networkoptimizer/data:/app/data \
  -v /opt/networkoptimizer/logs:/app/logs \
  --restart=unless-stopped \
  ghcr.io/ozark-connect/network-optimizer:latest
```

## Ligação ao UniFi

Feita pelo utilizador diretamente no painel (Settings → UniFi Connection), com uma conta **local**
dedicada criada na consola UniFi (não SSO — o próprio guia recomenda isto para permitir níveis de
acesso restritos: `Network View Only` + `Protect View Only` em vez de Super Admin, já que a maior
parte das funcionalidades são só de leitura). Confirmado a funcionar via logs do container:

```
Successfully authenticated with UniFi controller
Retrieved system info - Controller: UDM Pro HOLIVEIRA v10.6.101
```

**Nota de segurança:** acesso SSH à gateway é opcional no NetworkOptimizer, só necessário para
funcionalidades que *alteram* configuração (WAN steering, Adaptive SQM para bufferbloat, speed
test avançado). Decidido não ligar isso por agora — só a parte de leitura (auditoria, WiFi,
monitorização) está ativa.

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

## Estado (2026-09-16)

- ✅ LXC criada, Docker instalado, container saudável
- ✅ Ligado ao UniFi (UDM Pro HOLIVEIRA v10.6.101) com conta local dedicada
- ✅ InfluxDB configurado (buckets próprios criados pela app)
- ⏳ Auditoria de segurança inicial ainda por correr
- ⏳ Modem/ONT não configurado (IP de gestão não encontrado)
- ⏳ IP ainda em DHCP (192.168.2.224) — decisão de fixar/avançar a sério fica pendente do
  utilizador após explorar o painel
