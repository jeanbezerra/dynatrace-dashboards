# Instrumentação de servidor Linux com Dynatrace OneAgent e NetTracer

Guia operacional para ampliar a observabilidade de rede TCP de um servidor
Linux usando Dynatrace OneAgent, NetTracer e Network connection monitoring.
Revisado para **Latest Dynatrace** em 2026-08-03.

## Resultado esperado

Ao final da configuração, o Dynatrace deverá oferecer:

- métricas de bytes e pacotes transmitidos e recebidos por processo;
- pacotes TCP retransmitidos;
- RTT das conexões TCP;
- associação entre host, processo e, quando aplicável, contêiner;
- detalhes dos fluxos por IP de origem, IP de destino e porta do servidor;
- contagem de novas sessões, resets e timeouts;
- dados consultáveis em Dashboards e Notebooks com DQL.

## Como as peças se complementam

| Componente | Função |
| --- | --- |
| OneAgent | Descobre host, processos, interfaces e coleta métricas de infraestrutura e rede. |
| Network Agent | Coleta dados de rede de processos nativos em Linux, Windows e AIX. |
| NetTracer | Usa eBPF para observar eventos e métricas TCP em Linux, com foco especial em processos dentro de contêineres. |
| Network connection monitoring | Preserva detalhes de conexões e grava os fluxos como eventos no bucket Grail `default_network_flows`. |
| Grail/DQL | Permite consultar, correlacionar e visualizar métricas e eventos de rede. |

O NetTracer não é um agente instalado separadamente. O binário
`oneagentnettracer` já faz parte da instalação do OneAgent e é ativado pelas
configurações do Dynatrace.

> Para um servidor Linux sem contêineres, o módulo de rede normal do OneAgent
> continuará sendo a principal fonte. Em servidores com Docker, containerd ou
> Kubernetes, o NetTracer amplia a visibilidade das conexões dos contêineres.

## 1. Pré-requisitos

### Dynatrace

- Ambiente Dynatrace acessível pelo servidor diretamente ou por um Environment
  ActiveGate.
- Permissão para instalar OneAgent.
- Permissão para alterar configurações de infraestrutura e rede.
- Permissões de leitura de métricas e Events powered by Grail.
- OneAgent **1.337 ou superior** para Network connection monitoring.
- Modo de monitoramento **Full-Stack** ou **Infrastructure**. O modo Discovery
  não é suficiente para o NetTracer.

### Linux e NetTracer

O NetTracer requer kernel Linux 4.15 ou superior. As combinações explicitamente
testadas e indicadas pela documentação são:

| Distribuição | Arquitetura | Versão |
| --- | --- | --- |
| Red Hat Enterprise Linux | x86_64 | 8.0 ou superior |
| CentOS | x86_64 | 8.0 ou superior |
| Ubuntu | x86_64 | 18.04 LTS ou superior |

Para a instalação padrão não privilegiada do OneAgent, valide também:

- suporte a extended attributes no filesystem;
- pacote `libcap2`/`libcap` instalado;
- filesystem sem as opções `noexec` e `nosuid` no caminho do OneAgent;
- Linux Filesystem Capabilities habilitadas;
- autorização de segurança para uso de eBPF e das capabilities necessárias.

O `oneagentnettracer` usa normalmente:

- kernel 5.8 ou superior: `cap_bpf` e `cap_perfmon`;
- kernel anterior a 5.8, ou quando `cap_bpf` está bloqueada:
  `cap_sys_admin`;
- também `cap_dac_override`, `cap_sys_ptrace` e `cap_sys_resource`.

Não conceda essas capabilities manualmente antes da instalação. O instalador do
OneAgent aplica o conjunto compatível com o sistema. Use a lista acima para
revisão com a equipe de segurança e para diagnóstico.

## 2. Levantamento antes da instalação

Execute os comandos abaixo no servidor e registre a saída no chamado ou plano
de mudança.

```bash
uname -r
uname -m
cat /etc/os-release
```

Confirme o gerenciador de serviços e o modo de virtualização:

```bash
systemctl --version
systemd-detect-virt || true
```

Confirme filesystem e capabilities:

```bash
findmnt -no TARGET,FSTYPE,OPTIONS /
command -v getcap
cat /proc/cmdline
```

Em Debian/Ubuntu, confirme `libcap2-bin`:

```bash
dpkg-query -W libcap2 libcap2-bin
```

Em RHEL/CentOS, confirme `libcap`:

```bash
rpm -q libcap
```

Verifique a rota até o endpoint Dynatrace ou Environment ActiveGate definido
pela arquitetura. A URL correta deve ser obtida no comando de instalação
gerado pelo próprio ambiente.

```bash
getent hosts <dynatrace-ou-activegate>
curl -Iv https://<dynatrace-ou-activegate>/communication
```

Uma resposta HTTP não bem-sucedida ainda pode confirmar DNS, TCP e TLS. O que
não pode ocorrer é falha de resolução, timeout ou erro de confiança no
certificado esperado.

## 3. Instalar o OneAgent

No Dynatrace:

1. Abra **Discovery & Coverage**.
2. Selecione **Install > Install OneAgent**.
3. Escolha Linux e a arquitetura correta.
4. Selecione **Full-Stack** ou **Infrastructure**.
5. Defina, se aplicável, host group, network zone, proxy e nome do host.
6. Copie o comando de download e instalação gerado pelo ambiente.

O comando contém URLs e parâmetros específicos do tenant. Não substitua o
comando gerado por um exemplo estático deste documento.

Após baixar o instalador, um exemplo de execução é:

```bash
sudo /bin/sh ./Dynatrace-OneAgent-Linux.sh \
  --set-monitoring-mode=fullstack \
  --set-host-group=<grupo-do-host> \
  --set-network-zone=<zona-de-rede>
```

Remova os parâmetros que não se aplicam ao ambiente. O modo Full-Stack é
recomendado quando também se deseja tracing de aplicações; Infrastructure é
suficiente para a coleta de infraestrutura e NetTracer.

Proteja tokens, URLs assinadas e comandos de download como segredo. Evite
armazená-los em scripts, repositórios, tickets públicos ou histórico de shell.

### Servidor que já possui OneAgent

Confirme o modo atual:

```bash
cd /opt/dynatrace/oneagent/agent/tools
sudo ./oneagentctl --get-monitoring-mode
```

Se retornar `discovery`, altere para Full-Stack ou Infrastructure dentro de uma
janela de mudança. Por exemplo:

```bash
sudo ./oneagentctl \
  --set-monitoring-mode=infra-only \
  --restart-service
```

Alterar o modo reinicia o OneAgent e pode exigir reinício dos serviços
monitorados. Se o host já estiver em `fullstack`, não o reduza para
`infra-only` apenas para habilitar NetTracer.

## 4. Validar a instalação do OneAgent

```bash
sudo systemctl status oneagent --no-pager
sudo systemctl is-enabled oneagent
sudo systemctl is-active oneagent
```

Confirme os principais processos:

```bash
pgrep -af 'oneagentwatchdog|oneagentos|oneagentnetwork'
```

No Dynatrace, confirme que o host aparece em **Infrastructure & Operations >
Hosts** e que o OneAgent está saudável e atualizado.

> Processos de aplicação precisam ser reiniciados para receber módulos de
> deep monitoring. Isso não impede as métricas básicas de host, mas afeta a
> correlação completa de aplicação e processo.

## 5. Habilitar o NetTracer

### Piloto em um host específico

Comece por um host representativo antes de habilitar globalmente:

1. Abra **Hosts Classic** e selecione o host Linux.
2. Selecione **More (...) > Settings**.
3. Abra **NetTracer traffic**.
4. Ative **Enable NetTracer traffic network monitoring**.
5. Salve a configuração.

### Ambiente inteiro ou host group

Para habilitar por configuração central:

1. Abra **Settings > Network & Discovery > NetTracer traffic**.
2. Ative **Enable NetTracer traffic network monitoring**.
3. Se necessário, aplique a configuração ao host group antes de ampliar para o
   ambiente inteiro.

O schema usado pela Settings API é:

```text
builtin:nettracer.traffic
```

O atributo principal é `netTracer: true`, com escopos de ambiente, host group e
host. Para automação, consulte primeiro o schema do próprio tenant e não suponha
o formato de payload ou escopo.

## 6. Habilitar Network connection monitoring

Esta etapa é necessária para obter os registros detalhados por IP e porta no
Grail.

1. Abra **Settings > Collect and capture > Infrastructure > Network connection monitoring**.
2. Ative **Enable OneAgent network connection monitoring**.
3. Escolha o filtro de IP.
4. Escolha quais conexões serão reportadas.
5. Configure intervalo de agregação e rate limit.
6. Salve e aguarde pelo menos dois intervalos de coleta.

O schema usado pela Settings API é:

```text
builtin:network-connection-monitoring
```

### Filtro de IP

| Opção | Uso recomendado |
| --- | --- |
| All | Piloto controlado ou investigação curta em host de baixo volume. |
| Private traffic only | Comunicação interna entre aplicações e servidores. |
| Public traffic only | Dependências externas e tráfego de internet. |
| Custom inclusion | Melhor opção para investigar sub-redes ou parceiros específicos. |
| Custom exclusion | Remove redes irrelevantes, health checks ou proxies muito ruidosos. |

Inclusões e exclusões customizadas usam CIDR separado por vírgulas, por exemplo:

```text
10.20.0.0/16, 10.40.12.0/24, 2001:db8:1234::/48
```

### Conexões reportadas

| Opção | Comportamento |
| --- | --- |
| Critical connections | Padrão. Reporta principalmente timeouts de novas conexões e resets de conexões existentes. |
| All | Reporta conexões normais e críticas. Gera maior volume de eventos. |
| Custom | Reporta conexões que ultrapassam limiares de bytes, conectividade, retransmissão ou RTT. |

Para validar a instrumentação, use temporariamente **All** no host piloto ou uma
regra **Custom** que inclua o tráfego de teste. Se permanecer em Critical, uma
conexão TCP saudável pode não produzir registro, levando a um falso diagnóstico
de falha na instrumentação.

### Agregação e rate limit

- Intervalo padrão e recomendado: **1 minuto**.
- Intervalos maiores reduzem eventos, mas atrasam a visibilidade.
- Rate limit padrão: **100 registros de fluxo por host e por intervalo**.
- Em servidores muito ocupados, o limite pode ocultar pares de menor prioridade.
- Aumentar o limite ou reportar todas as conexões eleva o consumo de Events
  powered by Grail.

Comece com 1 minuto e 100 registros no piloto. Ajuste somente depois de medir o
volume e confirmar que os fluxos relevantes não estão sendo descartados.

Mantenha **Enable classic OneAgent process connection monitoring** desativado
em novas implementações. O modo clássico não é compatível com Grail e está
previsto para remoção.

## 7. Validar o NetTracer no Linux

Depois de ativar a configuração e gerar tráfego TCP, confirme o processo:

```bash
pgrep -af oneagentnettracer
```

Localize o binário instalado:

```bash
find /opt/dynatrace/oneagent -type f -name oneagentnettracer -print
```

Verifique as Linux capabilities sem modificá-las:

```bash
NETTRACER_BIN=$(find /opt/dynatrace/oneagent -type f -name oneagentnettracer -print -quit)
sudo getcap "$NETTRACER_BIN"
```

Confira mensagens recentes dos componentes do OneAgent:

```bash
sudo journalctl -u oneagent --since "30 minutes ago" --no-pager
sudo grep -RilE 'nettracer|ebpf|bpf' /var/log/dynatrace/oneagent 2>/dev/null
```

O diretório padrão de logs é `/var/log/dynatrace/oneagent`. Não altere
capabilities, políticas SELinux/AppArmor ou parâmetros de kernel com base apenas
em uma mensagem isolada; confirme a recomendação na documentação da versão do
OneAgent e com a equipe de segurança.

## 8. Gerar tráfego de validação

Use um endpoint autorizado e conhecido. Para testar uma conexão de saída:

```bash
curl -sS -o /dev/null \
  -w 'http_code=%{http_code} connect=%{time_connect}s total=%{time_total}s\n' \
  http://<servidor-de-teste>:<porta>/<health-check>
```

Para uma conexão recebida, execute o cliente em outro host contra uma porta já
exposta pela aplicação. Não abra portas novas nem provoque resets/timeouts em
produção apenas para validar a coleta.

Mantenha a chamada por dois ou três intervalos de um minuto. Portas em estado
listen sem conexão ativa não são suficientes: o NetTracer precisa observar uma
conexão TCP ativa para informar a porta.

## 9. Validar dados no Dynatrace com DQL

Substitua `<HOST-ID>` pelo Smartscape ID do host, por exemplo
`HOST-A1B2C3D4E5F67890`.

### Encontrar o ID do host

```dql
smartscapeNodes HOST
| filter name == "<nome-do-host>"
| fields id, name
```

### Confirmar eventos de fluxo do host

```dql
fetch events, bucket:{"default_network_flows"}
| filter dt.smartscape.host == toSmartscapeId("<HOST-ID>")
| sort timestamp desc
| limit 100
```

Se a consulta estiver vazia, confirme primeiro que **Reported connections** não
está limitado a Critical durante um teste composto somente por conexões
saudáveis.

### Resumo por processo

```dql
fetch events, bucket:{"default_network_flows"}
| filter dt.smartscape.host == toSmartscapeId("<HOST-ID>")
| summarize {
    Flows = count(),
    BytesRx = sum(network_flow.bytes.rx),
    BytesTx = sum(network_flow.bytes.tx),
    PacketsRx = sum(network_flow.packets.rx),
    PacketsTx = sum(network_flow.packets.tx),
    RetransmittedRx = sum(network_flow.packets.retransmitted.rx),
    RetransmittedTx = sum(network_flow.packets.retransmitted.tx),
    RttAvg = avg(network_flow.tcp.rtt)
  }, by:{dt.smartscape.process}
| fieldsAdd Process = getNodeName(dt.smartscape.process)
| sort Flows desc
```

### Pares e portas TCP mais ativos

```dql
fetch events, bucket:{"default_network_flows"}
| filter dt.smartscape.host == toSmartscapeId("<HOST-ID>")
| filter network_flow.network.transport == "TCP"
| summarize {
    Flows = count(),
    BytesRx = sum(network_flow.bytes.rx),
    BytesTx = sum(network_flow.bytes.tx),
    NewSessions = sum(network_flow.tcp.sessions.new),
    Resets = sum(network_flow.tcp.sessions.reset),
    Timeouts = sum(network_flow.tcp.sessions.timeout),
    RttAvg = avg(network_flow.tcp.rtt)
  }, by:{
    Process = dt.smartscape.process,
    ProcessIsServer = network_flow.process_is_server,
    SourceAddress = network_flow.source.address,
    DestinationAddress = network_flow.destination.address,
    DestinationPort = network_flow.destination.port
  }
| fieldsAdd ProcessName = getNodeName(Process)
| fieldsAdd TotalBytes = coalesce(BytesRx, 0) + coalesce(BytesTx, 0)
| sort TotalBytes desc
| limit 200
```

Por convenção, `source.address` representa o cliente que iniciou a conexão e
`destination.address`/`destination.port` representam o servidor TCP.
`network_flow.process_is_server` informa de qual lado está o processo observado.

### Conexões com problemas

```dql
fetch events, bucket:{"default_network_flows"}
| filter dt.smartscape.host == toSmartscapeId("<HOST-ID>")
| filter
    network_flow.tcp.sessions.reset > 0 or
    network_flow.tcp.sessions.timeout > 0 or
    network_flow.packets.retransmitted.rx > 0 or
    network_flow.packets.retransmitted.tx > 0
| fields
    timestamp,
    Process = getNodeName(dt.smartscape.process),
    network_flow.process_is_server,
    network_flow.source.address,
    network_flow.destination.address,
    network_flow.destination.port,
    network_flow.tcp.sessions.reset,
    network_flow.tcp.sessions.timeout,
    network_flow.packets.retransmitted.rx,
    network_flow.packets.retransmitted.tx,
    network_flow.tcp.rtt,
    network_flow.tcp.rtt.ack
| sort timestamp desc
| limit 200
```

### Métricas NetTracer no Grail

```dql
timeseries {
    BytesRx = sum(dt.process.nettracer.bytes_rx),
    BytesTx = sum(dt.process.nettracer.bytes_tx),
    PacketsRx = sum(dt.process.nettracer.pkts_rx),
    PacketsTx = sum(dt.process.nettracer.pkts_tx),
    RetransmittedPackets = sum(dt.process.nettracer.pkts_retr),
    Rtt = avg(dt.process.nettracer.rtt)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{dt.smartscape.host == toSmartscapeId("<HOST-ID>")},
  union:true
| fieldsAdd
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process)
| fieldsRemove dt.smartscape.host, dt.smartscape.process
```

### Descobrir quais métricas existem no host

```dql
metrics
| filter dt.smartscape.host == toSmartscapeId("<HOST-ID>")
| filter
    startsWith(metric.key, "dt.process.nettracer.") or
    startsWith(metric.key, "dt.process.network.") or
    startsWith(metric.key, "dt.host.net.")
| fields metric.key
| dedup metric.key
| sort metric.key asc
```

## 10. Critérios de aceite

Considere a instrumentação concluída quando todos os itens aplicáveis forem
verdadeiros:

- [ ] Host aparece no Dynatrace com OneAgent saudável.
- [ ] OneAgent está em Full-Stack ou Infrastructure.
- [ ] Versão do OneAgent é 1.337 ou superior.
- [ ] NetTracer está habilitado no escopo efetivo do host.
- [ ] Network connection monitoring está habilitado.
- [ ] `oneagentnettracer` executa após geração de tráfego, quando aplicável.
- [ ] Métricas `dt.process.nettracer.*` aparecem para processos em contêineres.
- [ ] Eventos aparecem em `default_network_flows`.
- [ ] Host e processo estão enriquecidos com `dt.smartscape.host` e
  `dt.smartscape.process`.
- [ ] IPs e portas esperados aparecem nos fluxos.
- [ ] RTT, pacotes, bytes, resets e timeouts apresentam valores coerentes.
- [ ] Volume de eventos e cardinalidade estão dentro do orçamento previsto.

## 11. Diagnóstico quando não há dados

### O host não aparece

- Confirme `systemctl is-active oneagent`.
- Valide DNS, proxy, TLS e rota ao Dynatrace/ActiveGate.
- Verifique a network zone configurada.
- Consulte **OneAgent Health** para incompatibilidade ou perda de heartbeat.

### O host aparece, mas não há NetTracer

- Confirme kernel 4.15 ou superior e arquitetura suportada.
- Confirme o modo Full-Stack ou Infrastructure.
- Verifique se a configuração efetiva foi sobrescrita no host ou host group.
- Valide `pgrep -af oneagentnettracer` depois de gerar tráfego.
- Inspecione as capabilities com `getcap`, sem alterá-las manualmente.
- Procure bloqueios de eBPF, SELinux, AppArmor ou política de hardening nos logs
  do sistema e do OneAgent.
- Em contêineres, confirme que o runtime e os namespaces estão visíveis para o
  OneAgent instalado no host.

### Há métricas, mas não há eventos em `default_network_flows`

- Confirme OneAgent 1.337 ou superior.
- Habilite Network connection monitoring, além do NetTracer.
- Durante o piloto, altere Reported connections de Critical para All ou Custom.
- Aguarde pelo menos dois intervalos de agregação.
- Verifique filtros de IP e CIDRs.
- Confirme se o rate limit não está sendo atingido.
- Verifique permissões para consultar Events powered by Grail.

### Faltam portas ou conexões

- Gere uma conexão TCP ativa; uma porta apenas em listen não basta.
- Confirme se o IP remoto passa pelo filtro configurado.
- Verifique o limite interno do NetTracer: somente 4096 conexões TCP podem ser
  rastreadas pelo módulo eBPF.
- Comunicação entre processos no mesmo host ou contêineres no mesmo nó pode
  gerar dois registros para o mesmo fluxo.
- Métricas de conectividade de sessão são reportadas somente para sessões de
  entrada na porta do servidor.

### Coletar diagnóstico para o suporte

```bash
cd /opt/dynatrace/oneagent/agent/tools
sudo ./oneagentctl --create-support-archive directory=/tmp days=1
```

O diretório informado precisa existir. O arquivo resultante contém configuração
e logs do OneAgent; trate-o como informação sensível e envie somente pelo canal
aprovado de suporte.

## 12. Segurança, privacidade e custo

- Network connection monitoring coleta endereços IP, portas e relacionamento
  com processos. Classifique esses dados de acordo com a política da empresa.
- Use Custom inclusion/exclusion para limitar redes sensíveis ou irrelevantes.
- A coleta não captura payload da aplicação; registra metadados e indicadores
  da conexão.
- O modo não privilegiado continua exigindo capabilities Linux específicas.
  Revise-as com segurança antes da implantação em massa.
- Reportar todas as conexões aumenta Events powered by Grail. Faça piloto,
  mensure volume e depois aplique filtros e limiares.
- Não habilite a coleta clássica de conexões em uma implantação nova baseada em
  Grail.

## 13. Estratégia recomendada de implantação

1. Selecione um host Linux não crítico, mas representativo.
2. Instale ou atualize o OneAgent para a versão atual.
3. Habilite NetTracer somente nesse host.
4. Habilite Network connection monitoring com filtro de IP restrito.
5. Use All por uma janela curta de validação.
6. Execute tráfego funcional conhecido.
7. Valide métricas, eventos, enriquecimento e volume.
8. Troque para Custom ou Critical conforme o caso de uso permanente.
9. Expanda para um host group.
10. Monitore custo e qualidade antes da ativação global.

## Referências oficiais

- [Install OneAgent on Linux](https://docs.dynatrace.com/docs/ingest-from/dynatrace-oneagent/installation-and-operation/linux/installation/install-oneagent-on-linux)
- [Customize OneAgent installation on Linux](https://docs.dynatrace.com/docs/ingest-from/dynatrace-oneagent/installation-and-operation/linux/installation/customize-oneagent-installation-on-linux)
- [OneAgent configuration via command-line interface](https://docs.dynatrace.com/docs/ingest-from/dynatrace-oneagent/oneagent-configuration-via-command-line-interface)
- [OneAgent non-privileged mode on Linux](https://docs.dynatrace.com/docs/ingest-from/dynatrace-oneagent/installation-and-operation/linux/installation/linux-non-privileged)
- [Extended network monitoring with NetTracer](https://docs.dynatrace.com/docs/observe/infrastructure-monitoring/networks/network-monitoring-with-nettracer)
- [OneAgent network connection monitoring](https://docs.dynatrace.com/docs/observe/infrastructure-observability/networks/network-devices/network-flows/oneagent-network-connection-monitoring)
- [Network connection monitoring schema](https://docs.dynatrace.com/docs/dynatrace-api/environment-api/settings/schemas/builtin-network-connection-monitoring)
- [NetTracer traffic schema](https://docs.dynatrace.com/docs/dynatrace-api/environment-api/settings/schemas/builtin-nettracer-traffic)
- [Built-in Metrics on Grail](https://docs.dynatrace.com/docs/analyze-explore-automate/metrics/built-in-metrics-on-grail)
