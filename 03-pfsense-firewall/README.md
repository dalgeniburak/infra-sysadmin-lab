# Módulo 03 — Firewall pfSense e Segmentação de Rede

## Objetivo

Instalar e configurar o pfSense CE como firewall/roteador do laboratório, segmentando a rede em WAN (rede física doméstica) e LAN (rede interna isolada), e habilitar acesso administrativo controlado da rede de gerenciamento até a LAN.

## Ambiente

- **VM:** `vm-pfsense-lab`
- **Specs:** 2 vCPU, 2GB RAM, 20GB disco (SCSI, VirtIO SCSI single), BIOS SeaBIOS
- **SO:** pfSense CE 2.8.1-RELEASE
- **Interfaces:**
  - WAN (em0) → bridge `vmbr0` → IP via DHCP (`192.168.100.x`)
  - LAN (em1) → bridge `vmbr1` (nova, criada sem porta física — rede isolada) → IP estático `192.168.10.1/24`, DHCP habilitado para clientes (`192.168.10.100–199`)

## Etapas realizadas

### 1. Criação da bridge de rede LAN (`vmbr1`)

Criada uma nova Linux Bridge no Proxmox (`vmbr1`), sem bridge port associado — ou seja, sem nenhuma interface física conectada, tornando-a uma rede puramente interna/virtual, compartilhada apenas entre VMs que a utilizem.

### 2. Criação da VM e instalação do pfSense CE

VM criada com duas interfaces de rede (`net0` → vmbr0/WAN, `net1` → vmbr1/LAN) e ISO do instalador Netgate montada.

**Problema encontrado (1):** interface WAN não conseguia obter conectividade real com a internet durante o instalador (verificação de licença/atualização falhava recorrentemente).

**Causa raiz:** a interface `net0` havia sido criada acidentalmente com uma **VLAN tag (tag=1)**, herdada de configuração anterior — a rede física doméstica não utiliza VLAN tagging, então os pacotes marcados eram descartados/ignorados pelo roteador físico.

**Correção:** remoção do campo VLAN Tag na configuração de hardware da interface `net0`.

Durante o instalador, optou-se pela edição **pfSense CE** (Community Edition, gratuita) ao invés de pfSense Plus, que exige assinatura paga — sem perda de funcionalidade relevante para o escopo do laboratório.

### 3. Atribuição de interfaces e definição de sub-redes

- WAN atribuída à `em0`, mantendo a mesma sub-rede da rede física doméstica (`192.168.100.0/24`).
- LAN atribuída à `em1`, com IP `192.168.10.1/24` — sub-rede deliberadamente **diferente** da WAN, para evitar conflito de roteamento (duas interfaces do mesmo firewall não podem pertencer à mesma sub-rede).

### 4. Acesso inicial à interface web (GUI)

**Problema encontrado (2):** o computador de administração está na rede `192.168.100.0/24` (WAN), sem rota nem conectividade direta com a rede LAN (`192.168.10.0/24`) recém-criada.

**Solução temporária:** criação de uma VM adicional (`vm-temp-lab`), com duas interfaces de rede (uma em cada bridge), servindo como "jumpbox" para o primeiro acesso à GUI do pfSense a partir da rede LAN.

Acesso inicial realizado via `https://192.168.10.1` a partir da jumpbox; senha padrão do usuário `admin` alterada no primeiro login.

### 5. Estudo de regras de firewall (Firewall → Rules)

Análise das regras padrão do pfSense:

- **Anti-Lockout Rule:** regra de proteção automática, garante acesso permanente à interface administrativa do próprio pfSense pela LAN, independente de outras regras — não editável diretamente.
- **Default allow LAN to any rule (IPv4/IPv6):** permite todo tráfego originado na LAN em direção a qualquer destino — reflete a prática padrão de firewalls de confiar no tráfego de saída da rede interna.
- Comportamento padrão da WAN: bloqueio total de tráfego de entrada, exceto o que for explicitamente liberado por regra.

### 6. Criação de regra de acesso administrativo (WAN → LAN)

Criada regra na aba **WAN** (não na LAN, já que o tráfego da rede de administração *entra* pelo pfSense através da interface WAN):

- Interface: WAN
- Protocolo: TCP
- Origem: Network `192.168.100.0/24`
- Destino: Network `192.168.10.0/24`
- Porta de destino: 443
- Descrição: "Permitir acesso admin a rede .100 LAN"

**Problema encontrado (3):** mesmo após salvar a regra, o acesso direto do computador físico à LAN continuava falhando (timeout).

**Diagnóstico em camadas:**
1. `tracert` a partir do computador físico mostrou que o tráfego seguia para o roteador doméstico, não para o pfSense — ausência de rota estática para a sub-rede `192.168.10.0/24`.
2. Adicionada rota estática temporária: `route add 192.168.10.0 mask 255.255.255.0 192.168.100.125` (IP da interface WAN do pfSense).
3. Mesmo com a rota corrigida, `Test-NetConnection` para a porta 443 continuava falhando.
4. Identificada opção **"Block private networks and loopback addresses"** habilitada na interface WAN — proteção padrão do pfSense que bloqueia automaticamente tráfego originado de redes privadas (RFC 1918), cenário pensado para WANs conectadas à internet pública, mas que neste laboratório bloqueava indevidamente o tráfego da rede doméstica privada.
5. Opção desmarcada, porém o acesso ainda falhava.
6. **Causa raiz final:** a alteração havia sido salva ("Save"), mas não aplicada ("Apply Changes") — no pfSense, mudanças de configuração de interface permanecem pendentes até confirmação explícita em um segundo passo.

**Resultado:** após aplicar a alteração corretamente, o acesso à interface web do pfSense a partir do computador físico (rede `192.168.100.0/24`) passou a funcionar normalmente, eliminando a necessidade da jumpbox para esse fim.

## Conceitos aplicados

- **Segmentação de rede:** separação de WAN e LAN em sub-redes distintas como prática de isolamento e controle de tráfego.
- **Regras de firewall como lógica condicional** (origem, destino, porta, protocolo, ação) e a diferença entre regras de proteção automática (Anti-Lockout) e regras gerais de política (Default allow LAN to any).
- **Direção de avaliação de regras:** uma regra que controla tráfego *entrando* por uma interface é configurada *naquela* interface de origem — não na interface de destino.
- **Roteamento:** necessidade de rota explícita para que um host saiba por qual gateway alcançar uma sub-rede que não é a sua local.
- **"Block private networks"** como proteção padrão de WAN, e por que ela pode ser inadequada em cenários de WAN sobre rede privada (típico de laboratórios domésticos).
- **Separação entre "salvar" e "aplicar"** configurações no pfSense — mudanças pendentes não têm efeito até confirmação explícita.
- Trade-off entre abrir acesso direto administrativo pela WAN (aceitável em ambiente controlado de laboratório) versus o uso de VPN como prática mais segura em ambientes de produção reais.

## Próximos passos

- Explorar VPN (ex.: OpenVPN/IPsec no pfSense) como alternativa mais segura de acesso administrativo remoto.
- Avaliar migração futura de DC01/DC02 para a rede LAN (`vmbr1`), protegidos atrás do firewall.

