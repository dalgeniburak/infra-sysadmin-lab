# Módulo 04 — Estrutura de OUs e Group Policy Objects (GPOs)

## Objetivo

Estruturar o domínio `lab.local` em Unidades Organizacionais (OUs), criar e aplicar uma Group Policy Object (GPO) de forma seletiva, e validar seu efeito prático em uma máquina cliente ingressada no domínio.

## Ambiente

- **Domain Controller:** `srv-dc01` (lab.local)
- **VM cliente de teste:** `USR-LAB-001` (Windows), ingressada no domínio

## Etapas realizadas

### 1. Criação da estrutura de OUs

Criadas três Unidades Organizacionais na raiz do domínio, via **Active Directory Users and Computers**:

- `Financeiro`
- `TI`
- `Diretoria`

Mantida a opção padrão **"Protect container from accidental deletion"** habilitada em cada OU, evitando exclusão acidental de uma unidade inteira (e de todos os objetos nela contidos).

### 2. Criação de usuário de teste

Criado o usuário `usuario.financeiro` diretamente dentro da OU `Financeiro`, para servir de base ao teste de política.

**Conceito consolidado:** diferença entre **OU** (localização organizacional do objeto no AD, usada para aplicar GPOs e delegar administração) e **grupo** (conjunto de permissões/acessos que um usuário carrega, independente de sua OU). Um objeto pertence a uma única OU, mas pode integrar múltiplos grupos.

### 3. Criação da GPO

GPO criada e vinculada diretamente à OU `Financeiro`, via **Group Policy Management (gpmc.msc)**:

- Nome: `GPO-Financeiro-Bloqueio-PainelControle`
- Configuração: **User Configuration → Policies → Administrative Templates → Control Panel → "Prohibit access to Control Panel and PC settings"** → Enabled

### 4. Ingresso de máquina cliente no domínio

VM cliente `USR-LAB-001` ingressada no domínio `lab.local`.

**Problema encontrado:** erro "Não foi possível contatar um Controlador de Domínio (AD DC) para o domínio 'lab.local'" — mesma causa raiz já identificada no módulo do DC02: configuração de DNS da máquina não apontava para o DC01 (`192.168.100.201`).

**Correção:** ajuste do DNS preferencial da máquina cliente, seguido de novo ingresso no domínio, bem-sucedido.

### 5. Posicionamento do objeto computador na OU correta

Por padrão, o objeto computador é criado na pasta genérica `Computers` ao ingressar no domínio — movido manualmente para dentro da OU `Financeiro` (em uma sub-OU `Computers` própria), via **Active Directory Users and Computers → Move**.

**Conceito consolidado:** herança de política — uma GPO vinculada a uma OU também se aplica automaticamente às sub-OUs contidas nela, sem necessidade de vínculo adicional.

### 6. Validação prática

- Login na VM cliente com o usuário `usuario.financeiro`.
- Execução de `gpupdate /force` para forçar a aplicação imediata da política, sem aguardar o ciclo automático padrão (~90–120 minutos).
- Tentativa de abrir o Painel de Controle: acesso bloqueado com sucesso, confirmando a aplicação efetiva da GPO.

## Conceitos aplicados

- Estrutura de OUs como base organizacional do Active Directory, distinta do conceito de grupos de segurança.
- Vínculo de GPOs a OUs específicas, permitindo aplicação seletiva de políticas por área/departamento.
- Herança de política entre OU pai e sub-OUs.
- Dependência de DNS correto para ingresso de máquinas no domínio (mesma causa raiz observada anteriormente no módulo do DC02).
- Ciclo de atualização de política (`gpupdate /force`) como ferramenta de validação e diagnóstico.

## Próximos passos

- Testar GPOs adicionais (ex.: política de senha, scripts de logon, mapeamento de unidade de rede).
- Validar escopo/isolamento da política testando um usuário da OU `TI` na mesma máquina, confirmando que o bloqueio não se aplica fora do escopo pretendido.

## Screenshots

_Ver pasta [`screenshots/`](./screenshots/)._
