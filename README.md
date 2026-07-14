# FortiGate Link Monitor — Links WAN como hosts no Zabbix

Templates Zabbix que transformam cada **Link Monitor / Performance SLA** de um firewall FortiGate em um **host lógico individual** no Zabbix, criado e mantido automaticamente via descoberta de baixo nível (LLD) com *host prototypes*.

Cada circuito WAN passa a existir como um objeto próprio no Zabbix — pronto para uso em mapas de NOC, dashboards por circuito e filtros de alerta — usando **exatamente o mesmo critério de saúde que o FortiGate usa para decisões de failover e SD-WAN**, sem necessidade de criar hosts manualmente e sem gerar tráfego ICMP adicional na rede.

## Como funciona

```
┌──────────────────────┐   SNMP (fgLinkMonitor MIB)   ┌─────────────────────────┐
│      FortiGate       │ ◄─────────────────────────── │         Zabbix          │
│                      │                              │                         │
│  Link Monitor "A" ───┼──► descoberta LLD ──────────►│  Host "WAN - A"         │
│  Link Monitor "B" ───┼──► (host prototype) ────────►│  Host "WAN - B"         │
│  Link Monitor "C" ───┼─────────────────────────────►│  Host "WAN - C"         │
└──────────────────────┘                              └─────────────────────────┘
```

1. O template **FortiGate Link WAN host discovery** é vinculado ao host do FortiGate. Sua regra de descoberta percorre via SNMP a tabela `fgLinkMonitorName` (OID `1.3.6.1.4.1.12356.101.4.8.2.1.2`).
2. Para cada Link Monitor encontrado, um *host prototype* cria automaticamente um host chamado `WAN - <nome do link>` no grupo **Links WAN**.
3. Cada host descoberto **herda a interface SNMP do FortiGate pai** (as consultas continuam indo para o IP do firewall) e recebe a macro `{$LINKMON.INDEX}` com o índice SNMP daquele link.
4. O template **Link WAN by FortiGate SNMP**, aplicado automaticamente aos hosts descobertos, coleta as métricas daquele circuito usando OIDs no formato `1.3.6.1.4.1.12356.101.4.8.2.1.X.{$LINKMON.INDEX}`.

Como a fonte dos dados é o próprio Link Monitor do FortiGate, o estado exibido no Zabbix é o mesmo que o firewall usa internamente — se o Zabbix mostra o link como *Dead*, é porque o FortiGate já o tirou de operação.

## Requisitos

| Requisito | Detalhe |
|---|---|
| Zabbix | 6.0 LTS ou superior (formato de exportação 6.0, aceito também por 6.4 e 7.x) |
| FortiGate | FortiOS com Link Monitor / Performance SLA configurado e MIB `fgLinkMonitor` disponível |
| SNMP | v2c ou v3 habilitado no FortiGate e host do FortiGate já monitorado no Zabbix com interface SNMP funcional |

Para validar se o seu FortiGate expõe os OIDs necessários:

```bash
snmpwalk -v2c -c <community> <ip-do-fortigate> 1.3.6.1.4.1.12356.101.4.8.2.1.2
```

Se retornar os nomes dos seus Link Monitors, o template vai funcionar. Se retornar `No Such Instance`, não há Link Monitor / Performance SLA configurado nesse equipamento (ou o VDOM consultado não os possui).

## Instalação

1. Baixe o arquivo `fortigate-linkwan-hosts.yaml`.
2. No Zabbix, acesse **Coleta de dados → Templates → Importar** e selecione o arquivo. Mantenha as opções padrão ("Criar novo" e "Atualizar existente" marcados) e **não marque nenhuma opção "Excluir ausentes"**.
3. Vincule o template **FortiGate Link WAN host discovery** ao host do FortiGate (mantendo os templates que já estão nele).
4. A descoberta roda a cada 1 hora. Para executar imediatamente: **Coleta de dados → Hosts → (FortiGate) → Regras de descoberta → Link WAN host discovery → Executar agora**.
5. Os hosts `WAN - <nome>` aparecerão no grupo **Links WAN**.

## O que é criado

### Itens (por circuito)

| Item | Chave | OID (sufixo) | Coleta |
|---|---|---|---|
| Estado do link (0 = Alive / 1 = Dead) | `linkwan.state` | `.3` | 1m |
| Latência (ms) | `linkwan.latency` | `.4` | 1m |
| Jitter (ms) | `linkwan.jitter` | `.5` | 1m |
| Perda de pacotes (%) | `linkwan.loss` | `.8` | 1m |
| Pacotes enviados por segundo | `linkwan.sent` | `.6` | 1m |
| Pacotes recebidos por segundo | `linkwan.received` | `.7` | 1m |
| Pacotes fora de sequência por segundo | `linkwan.outofseq` | `.13` | 1m |
| Banda disponível de entrada (Mbps) | `linkwan.bandwidth.in` | `.10` | 5m |
| Banda disponível de saída (Mbps) | `linkwan.bandwidth.out` | `.11` | 5m |
| Banda disponível bi-direção (Mbps) | `linkwan.bandwidth.bi` | `.12` | 5m |

Latência, jitter e perda possuem preprocessing com expressão regular que extrai apenas o valor numérico — necessário porque algumas versões do FortiOS retornam esses valores como texto com sufixo (ex.: `"100.000%"`). Valores vazios ou não numéricos (comum em links mortos) são descartados silenciosamente, sem colocar o item em estado de erro.

### Triggers

| Trigger | Severidade | Condição |
|---|---|---|
| Link WAN indisponível: Link Monitor reporta DEAD | Alta | `last(state) = 1` |
| Latência alta no link WAN | Aviso | `min(5m) > {$LINKWAN.LATENCY.MAX}` |
| Perda de pacotes alta no link WAN | Média | `min(5m) > {$LINKWAN.LOSS.MAX}` |

As triggers de latência e perda **dependem** da trigger de indisponibilidade: quando o link cai, você recebe apenas o alerta de DEAD, sem ruído duplicado.

### Macros configuráveis

| Macro | Padrão | Descrição |
|---|---|---|
| `{$LINKWAN.LATENCY.MAX}` | `200` | Latência máxima aceitável (ms) antes de alertar |
| `{$LINKWAN.LOSS.MAX}` | `10` | Perda máxima aceitável (%) antes de alertar |
| `{$LINKMON.INDEX}` | — | Índice SNMP do link; preenchido automaticamente pelo host prototype, não altere manualmente |

Os limiares podem ser sobrescritos globalmente (no template), por circuito (no host descoberto, via herança de macro) ou por grupo.

### Tags

Todos os eventos saem identificados em três níveis:

- **Host descoberto:** `tipo: link-wan` e `link: <nome do link monitor>`
- **Itens:** `component: link-wan`
- **Triggers:** `scope: availability` ou `scope: performance`, `component: link-wan` e `circuito: <nome do host>`

Isso permite, por exemplo, rotear ações de alerta apenas para eventos com `component = link-wan`, ou filtrar o widget de Problemas por `scope = availability`.

## Limitações e observações

- **Nomes de host precisam ser únicos no Zabbix.** Se dois FortiGates tiverem Link Monitors com o mesmo nome (ex.: dois sites com um monitor chamado `VIVO`), a descoberta do segundo falhará. Solução: renomear os Link Monitors no FortiGate com nomes únicos (ex.: `VIVO-MTZ`, `VIVO-SP`) ou clonar o template de descoberta ajustando o prefixo do host prototype por site.
- Os hosts descobertos são gerenciados pelo LLD: se um Link Monitor for removido do FortiGate, o host correspondente é removido após o período de `lifetime` da regra (7 dias por padrão).
- O template não substitui a monitoração do próprio FortiGate (CPU, memória, interfaces, VPN) — use-o em conjunto com o template SNMP do firewall.
- Um Link Monitor com 100% de perda e estado *Dead* é um diagnóstico do próprio FortiGate; verifique se o destino monitorado no Performance SLA ainda é válido antes de suspeitar do template.

## Estrutura do repositório

| Arquivo | Descrição |
|---|---|
| `fortigate-linkwan-hosts.yaml` | Templates de descoberta de hosts + monitoração por circuito (este README) |
| `fortigate-linkmon-dashboard.yaml` | (Opcional) Dashboard de template com gráficos por circuito para o template FortiGate existente |

## Referência da MIB

Tabela `fgLinkMonitorTable` — base `1.3.6.1.4.1.12356.101.4.8.2.1`:

`.2` Name · `.3` State · `.4` Latency · `.5` Jitter · `.6` PacketSend · `.7` PacketRecv · `.8` PacketLoss · `.9` Vdom · `.10` BandwidthIn · `.11` BandwidthOut · `.12` BandwidthBi · `.13` OutofSeq · `.14` Server · `.15` Protocol

## Licença

Distribuído sob a licença MIT. Sinta-se livre para usar, modificar e contribuir.
