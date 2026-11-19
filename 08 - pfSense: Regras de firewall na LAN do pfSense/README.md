
# Regras de firewall na LAN do pfSense: bloqueio, ordem das regras e leitura de logs

## Objetivo

Entender como o pfSense decide o que passa e o que é barrado, criando uma regra de bloqueio na LAN, provando que ela funciona, quebrando-a de propósito (alterando a ordem) e usando os logs para diagnosticar o resultado.

## Ambiente

| Item | Valor |
|---|---|
| Firewall | pfSense Community Edition 2.9.0 (VM no Proxmox) |
| WAN | vtnet0, 192.168.100.133/24 (DHCP) |
| LAN | vtnet1, 192.168.1.1/24, DHCP de .100 a .199 |
| Cliente de teste | VM Windows atrás do pfSense, 192.168.1.100 |

## Parte 0: revisão do Port Forward (módulo anterior)

Antes do lab, foram revisados os conceitos do Port Forward (WAN 192.168.100.133:3389 → VM 192.168.1.100, origem restrita ao PC 192.168.100.110).

- **Regra de NAT:** apenas traduz o endereço de destino (DNAT). Não decide se o tráfego pode passar.
- **Regra de firewall:** decide se o pacote é permitido ou bloqueado. Sem ela, o pacote traduzido seria derrubado pela negação padrão da WAN.
- **Ordem de processamento na entrada:** o NAT acontece antes do firewall. Por isso a regra da WAN usa como destino o IP **interno** da VM, e não o IP da WAN.
- **Restringir a origem:** reduz a superfície de exposição. Só o IP autorizado consegue usar a porta aberta.
- **Block private networks:** bloqueia na WAN tráfego com origem privada (10.x, 172.16-31.x, 192.168.x), que em uma WAN ligada à internet seria spoofing. Foi desmarcado no lab porque a "WAN" do lab é a rede doméstica, que também é privada. Em produção, deve ficar marcado.
- **Firewall stateful:** a conexão permitida entra na tabela de estados, então a resposta volta sem precisar de outra regra.

## Parte 1: teste de base ("antes")

Na VM cliente, antes de qualquer regra restritiva:

```powershell
ping 8.8.8.8
```

Resultado: 4 enviados, 4 recebidos, 0% de perda. O Google abriu no navegador. Isso confirmou que a VM sai pela LAN, passa pelo pfSense, alcança a internet e resolve nomes (DNS).

## Parte 2: regra de bloqueio de ping

**Firewall > Rules > LAN > Add (seta para cima, para criar no topo):**

| Campo | Valor |
|---|---|
| Action | Block |
| Interface | LAN |
| Address Family | IPv4 |
| Protocol | ICMP (any) |
| Source | LAN subnets |
| Destination | Single host: 8.8.8.8 |
| Log | Marcado |
| Description | Teste - bloqueio de ping para 8.8.8.8 |

Depois: **Save** e **Apply Changes** (sem o Apply a regra não entra em vigor).

**Resultado:**

- `ping 8.8.8.8` retornou "Esgotado o tempo limite do pedido" (100% de perda).
- O Google continuou abrindo no navegador.

**Por que o Google continuou abrindo:** a regra só casa com pacotes que combinam **todos** os campos: protocolo ICMP e destino 8.8.8.8. O navegador usa TCP 443 (HTTPS) para outro IP, e o DNS usa a porta 53. Esses pacotes não casam com a regra de bloqueio, então caem na "Default allow LAN to any rule".

## Parte 3: ordem das regras ("quebrar")

O pfSense avalia as regras **de cima para baixo** e a **primeira que casa decide** o destino do pacote. As demais não são consultadas.

**Experimento:** a regra de bloqueio foi arrastada para **baixo** da "Default allow LAN to any rule", seguida de Save e Apply Changes.

**Resultado:** o ping voltou a responder (4 recebidos, 0% de perda). A "Default allow" libera tudo e vem primeiro, então o pacote é liberado antes de chegar ao bloqueio. O contador da regra de bloqueio parou de aumentar.

**Lição:** um bloqueio abaixo de uma regra de liberação ampla nunca é executado. É um dos erros mais comuns em firewalls de produção.

## Parte 4: leitura dos logs

**Status > System Logs > Firewall.** Cada linha responde a seis perguntas:

| Coluna | O que informa |
|---|---|
| Action | X vermelho = bloqueado; check verde = liberado |
| Interface | Por onde o pacote entrou (WAN ou LAN) |
| **Rule** | **Qual regra decidiu** (a coluna mais importante) |
| Source | Origem (IP:porta) |
| Destination | Destino (IP:porta) |
| Protocol | TCP, UDP ou ICMP |

**Filtro:** o ícone de funil abre o "Advanced Log Filter". Filtrando por Destination `8.8.8.8` apareceram 8 entradas bloqueadas, todas com a Rule "Teste - bloqueio de ping para 8.8.8.8", protocolo ICMP e origem 192.168.1.100. Isso corresponde a dois testes de 4 pacotes e ao contador da regra (480 B = 8 pacotes de 60 bytes). Log, contador e teste confirmam a mesma coisa.

**Observação sobre horário:** o pfSense exibe o horário em UTC, três horas à frente do horário de Brasília.

### Interpretação de entradas encontradas no log

| Entrada | Interpretação |
|---|---|
| `WAN \| Default deny \| 192.168.100.110:60827 → 192.168.100.133:3389 \| UDP` | O cliente RDP tenta também UDP na 3389. O Port Forward libera só TCP, então o UDP é barrado. O RDP funciona normalmente pelo TCP. |
| Destinos 224.0.0.251:5353 e 239.255.255.250:1900 | Ruído normal de redes domésticas (mDNS e SSDP). Pode ser ignorado. |
| `block bogon IPv6 networks from WAN` | Ruído de IPv6 na WAN. |
| `LAN \| 192.168.1.100 → 172.171.87.38:443 \| TCP:FA / TCP:RA` | Pacotes de encerramento de conexão HTTPS que chegaram após o pfSense fechar o estado. Inofensivo. |

## Método de troubleshooting

Quando algo não conecta, a coluna **Rule** do log indica quem decidiu:

- **Nome de uma regra criada por você:** a regra casou. Verifique se a ação (Block/Pass) e a posição estão como pretendido.
- **Default deny:** nenhuma regra liberou o pacote. Compare o pacote do log com a regra esperada, campo por campo (origem, destino, porta, protocolo, interface). O campo que divergir mostra o que corrigir. Também verifique se o NAT aponta para o IP correto.
- **Sem entrada no log:** o pacote pode nem ter chegado ao firewall (rota, gateway da VM, cabo virtual/bridge).

Checklist adicional para o Port Forward: IP de origem ainda é o do PC autorizado?; IP da VM mudou (DHCP)? Considere uma reserva de IP (static mapping); serviço ativo na VM e firewall do Windows liberando a porta?; gateway da VM aponta para o pfSense?

## Aprendizados

1. NAT traduz, firewall decide; os dois trabalham juntos.
2. Uma regra só afeta o tráfego que combina com todos os seus campos.
3. A ordem das regras é decisiva: a primeira que casa vence.
4. O log (coluna Rule) é a principal ferramenta de diagnóstico.
5. Provar que uma regra funciona exige teste antes e depois, com log e contador como evidência.

## Próximos passos

- Reserva de IP (static mapping) para a VM cliente.
- Regras por porta/serviço (liberar só DNS e HTTPS, bloquear o resto).
- Bloqueio por alias (lista de IPs ou sites) e uso de agendamento (Schedule).
- Segmentação com mais de uma rede interna (VLANs).
