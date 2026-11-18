# 07 – pfSense: Port Forward para acesso remoto (RDP)

**Data:** 17/11/2026
**Ambiente:** laboratório em Proxmox, firewall pfSense 2.9.0 (VM) e uma VM Windows (vm-win-lab) atrás dele.

## 1. Objetivo

Criar no pfSense um redirecionamento de porta (Port Forward) com a regra de firewall correspondente, para acessar remotamente a vm-win-lab, que fica na LAN interna, a partir de um PC na rede da WAN. O ponto de partida era o VNC (porta 5900). Como a versão instalada do VNC não conectou, o acesso foi trocado para RDP (porta 3389), reaproveitando o mesmo Port Forward.

## 2. Topologia

| Elemento | Endereço | Observação |
|---|---|---|
| PC de acesso | 192.168.100.110 | Rede da WAN do pfSense |
| pfSense – WAN (vtnet0) | 192.168.100.133 | Endereço onde o Port Forward responde |
| pfSense – LAN (vtnet1) | 192.168.1.1/24 | Rede interna do lab (vmbr1), sem porta física |
| vm-win-lab | 192.168.1.100 | Destino final do acesso remoto |

Fluxo: `PC (192.168.100.110) → WAN do pfSense (192.168.100.133:3389) → vm-win-lab (192.168.1.100:3389)`

## 3. Conceitos envolvidos

**Port Forward (NAT de destino).** Traduz o destino do pacote. Tudo que chega na WAN do pfSense na porta 3389 é reescrito para o IP e a porta da VM interna. Ele responde sobre onde o tráfego vai.

**Regra de firewall (filtro).** Decide se o pacote pode passar. Na WAN o pfSense bloqueia por padrão, então sem uma regra de permissão o pacote é descartado mesmo depois de traduzido. O pfSense avalia a regra depois da tradução: por isso a regra associada mostra como destino o IP da VM (192.168.1.100) e a porta 3389, e não o endereço da WAN.

Os dois trabalham juntos: o NAT diz para onde o pacote vai, e a regra diz se ele pode passar. Ao marcar *Add associated filter rule*, o pfSense cria a regra de filtro automaticamente e a mantém sincronizada com o Port Forward.

**Block private networks.** Opção da interface WAN que descarta pacotes cuja origem seja endereço privado (RFC 1918). Em uma WAN de internet ela serve como proteção contra pacotes com origem forjada, porque origens privadas não deveriam chegar da internet. Neste lab, o PC de acesso está em uma rede privada (192.168.100.0/24), então a opção precisou ser desmarcada. Ela não abre nenhuma porta: só deixa de descartar origens privadas.

## 4. Passo a passo

1. **Liberar origens privadas na WAN:** em *Interfaces > WAN > Reserved Networks*, desmarcar *Block private networks and loopback addresses*. Salvar e aplicar. Manter *Block bogon networks* marcado.
2. **Criar o Port Forward:** em *Firewall > NAT > Port Forward > Add*:
   - Interface: WAN; Protocol: TCP
   - Source: Address or Alias, com o IP do PC (192.168.100.110)
   - Destination: WAN address; porta 3389 (de início e de fim)
   - Redirect target IP: 192.168.1.100 (a VM, não o pfSense)
   - Redirect target port: 3389
   - Filter rule association: *Add associated filter rule*
3. **Salvar e aplicar as mudanças.**
4. **Preparar a VM:** ativar a Área de Trabalho Remota em *Configurações > Sistema* e garantir que o usuário tenha senha.
5. **Conectar:** no PC, abrir a Conexão de Área de Trabalho Remota e usar `192.168.100.133` como computador.

## 5. Validação

- `Test-NetConnection 192.168.100.133 -Port 5900` (primeira versão, com VNC) retornou `TcpTestSucceeded : True`, provando que o pacote atravessava o pfSense e chegava na VM.
- Em *Firewall > Rules > WAN*, a regra associada ao Port Forward apareceu com origem 192.168.100.110 e destino 192.168.1.100, e o contador de tráfego mostrou bytes passando por ela. Se estivesse bloqueado, o contador ficaria zerado.
- A conexão RDP em `192.168.100.133` foi estabelecida com sucesso.

## 6. Troubleshooting

| Problema | Causa | Solução |
|---|---|---|
| Redirect target IP apontava para 192.168.1.1 | Foi usado o IP do próprio pfSense, que não tem serviço de acesso remoto | Trocar pelo IP da VM (192.168.1.100) |
| Campos da regra de filtro travados para edição | A regra associada é criada e gerenciada pelo Port Forward | Editar pelo Port Forward em *Firewall > NAT*; a regra se atualiza sozinha |
| VNC não conectou mesmo com a porta chegando na VM | Problema do lado do programa de VNC instalado, e não do firewall (o teste de porta deu True) | Trocar o acesso para RDP e ajustar o Port Forward para a porta 3389 |
| RDP falhou ao conectar em 192.168.100.100 | Esse endereço é outro equipamento da rede, e não o pfSense | Usar a WAN do pfSense (192.168.100.133), que é onde o Port Forward responde |

## 7. Segurança

- A origem do Port Forward está restrita ao IP do PC de acesso, o que impede que outros equipamentos da rede usem a regra.
- RDP e VNC expostos diretamente na internet são alvos frequentes de ataque. Este desenho é válido apenas para o laboratório. Em um cenário real, o acesso deve passar por VPN.
- Em uma WAN de internet, manter *Block private networks* marcado.
- Usar senha forte no usuário da VM.

## 8. Aprendizados

- O Port Forward só responde no endereço da WAN do pfSense. A VM interna não é acessada diretamente de fora.
- NAT traduz o destino e a regra de firewall autoriza. Os dois precisam existir.
- Desmarcar *Block private networks* no lab é uma exceção por causa do endereçamento. Não é um padrão para WAN real.
- O log de firewall e o contador de bytes da regra mostram se o tráfego está passando ou sendo bloqueado.
- Um teste de porta bem-sucedido separa o problema do firewall do problema do serviço no destino.

## 9. Próximos passos

- Registrar o tráfego da regra e consultar *Status > System Logs > Firewall*.
- Criar regras na LAN para controlar o que a VM pode acessar.
- Substituir o Port Forward por uma VPN (OpenVPN ou WireGuard no pfSense).

## Prints sugeridos para anexar

1. Interfaces > WAN > Reserved Networks, com *Block private networks* desmarcado.
2. Tela de edição do Port Forward, com origem, destino e redirecionamento.
3. Firewall > Rules > WAN, mostrando a regra associada.
4. Resultado do `Test-NetConnection` com `TcpTestSucceeded : True`.
5. Conexão RDP aberta na VM.
