# Módulo 06 — Delegação de Permissões Administrativas (Delegation of Control)

## Objetivo

Conceder a um usuário comum (`usuario.ti`) a capacidade de resetar senhas e gerenciar membership de grupos apenas dentro da OU Financeiro, sem torná-lo Administrador de Domínio — simulando um cenário real de help desk com autonomia limitada.

## Ambiente

- **Domain Controller:** `srv-dc01` (lab.local)
- **VM cliente de teste:** `USR-LAB-001`
- **Escopo da delegação:** OU `Financeiro`
- **Usuário delegado:** `usuario.ti`

## Etapas realizadas

### 1. Delegação de controle via Delegation of Control Wizard

No **Active Directory Users and Computers**, delegação aplicada especificamente à OU `Financeiro` (não ao domínio inteiro), concedendo ao usuário `usuario.ti` as seguintes tarefas:

- **Reset user passwords and force password change at next logon**
- **Modify the membership of a group**

O assistente confirma o escopo restrito antes de aplicar (`lab.local/Financeiro`), garantindo que a permissão não se estenda a outras OUs (TI, Diretoria).

**Conceito consolidado:** delegação de controle no AD opera em granularidade de OU — é possível conceder tarefas administrativas específicas e limitadas, evitando a prática comum (e arriscada) de adicionar usuários de suporte diretamente ao grupo de Administradores de Domínio.

### 2. Tentativa inicial de validação — bloqueio de logon local no DC

**Problema encontrado:** tentativa de testar a delegação logando como `usuario.ti` diretamente no Domain Controller (`srv-dc01`), inclusive via `runas`, falhou com o erro:

> "1385: Logon failure: the user has not been granted the requested logon type at this computer."

**Causa raiz:** por padrão, apenas contas com privilégios elevados (Administradores de Domínio, Operadores de Servidor, etc.) possuem o direito de logon local em um Domain Controller — uma proteção de segurança padrão, já que o DC é o componente mais crítico da infraestrutura de identidade da rede. Delegação de tarefas no AD é conceito distinto do direito de logon local em uma máquina.

**Correção de abordagem:** validação realizada a partir de uma máquina cliente (`USR-LAB-001`), ambiente correto para testar permissões delegadas via ferramentas RSAT.

### 3. Instalação de RSAT na máquina cliente

**Problema encontrado (1):** tentativa de instalar o RSAT (Rsat.ActiveDirectory.DS-LDS.Tools) logada como `usuario.ti` falhou por falta de permissão de administrador **local** na máquina — reforçando a distinção entre permissões delegadas no AD e privilégios administrativos locais de uma estação de trabalho.

**Correção:** instalação realizada com conta de Administrador local.

**Problema encontrado (2):** instalação falhou por espaço em disco insuficiente.

**Diagnóstico e correção, em camadas:**
1. Adicionado um disco extra à VM no Proxmox — porém identificado que, por engano, foi criado um **novo disco** (Disco 1) em vez de expandir o disco original (Disco 0, onde reside o volume C:).
2. Disco extra removido do Proxmox (Detach seguido de Remove), evitando desperdício de espaço de armazenamento.
3. Disco original (Disco 0) redimensionado corretamente no Proxmox.
4. Ainda assim, a extensão do volume C: não era oferecida no Gerenciamento de Disco do Windows — identificado que uma **partição de recuperação** (532 MB) estava posicionada entre o volume C: e o espaço não alocado, impedindo a extensão direta (o Windows só estende um volume usando espaço não alocado imediatamente adjacente).
5. A interface gráfica não permitia excluir a partição de recuperação diretamente (proteção padrão do Windows).
6. Partição de recuperação removida via **DiskPart** (`select partition`, `delete partition override`), liberando espaço contíguo ao volume C:.
7. Volume C: estendido com sucesso, absorvendo o espaço liberado.

**Resultado:** instalação do RSAT (`Rsat.ActiveDirectory.DS-LDS.Tools` e `Rsat.ServerManager.Tools`) concluída com sucesso.

### 4. Validação final da delegação

Login na máquina cliente como `usuario.ti`, abertura do **Active Directory Users and Computers** via RSAT:

- **Teste positivo:** reset de senha do usuário `usuario.financeiro` (membro da OU Financeiro) — bem-sucedido, confirmado pela mensagem "A senha para usuario.financeiro foi alterada".
- **Teste negativo (isolamento):** tentativa de reset de senha do próprio `usuario.ti` (membro da OU TI, fora do escopo delegado) — corretamente bloqueada, com a mensagem "O Windows não pode concluir a alteração da senha... Acesso negado".

Os dois testes em conjunto confirmam que a delegação está funcionando e corretamente restrita ao escopo pretendido, sem vazamento de permissão para outras OUs.

## Conceitos aplicados

- Delegação de controle no AD como alternativa a conceder privilégios de Administrador de Domínio para tarefas administrativas pontuais e restritas por OU.
- Diferença entre delegação de tarefas no AD e o direito de logon local em uma máquina (DC ou estação de trabalho) — são camadas de permissão independentes.
- Diferença entre privilégios administrativos de domínio e privilégios administrativos locais de uma máquina específica.
- Restrições do Windows ao redimensionamento de partições (necessidade de espaço não alocado contíguo) e uso do DiskPart para operações que a interface gráfica bloqueia por segurança (ex.: exclusão de partição de recuperação).
- Importância de validar não apenas o caminho de sucesso ("consegue fazer X"), mas também o limite da permissão ("não consegue fazer Y fora do escopo") — validação dupla como prática de teste mais rigorosa.

## Próximos passos

- Explorar scripts de logon como forma adicional de automação via GPO.
- Avaliar delegação de outras tarefas comuns de help desk (ex.: desbloqueio de conta, edição de atributos específicos).

## Screenshots

_Ver pasta [`screenshots/`](./screenshots/)._
