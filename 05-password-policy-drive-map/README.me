# Módulo 05 — Política de Senha e Mapeamento de Rede via GPO

## Objetivo

Reforçar a política de senha do domínio segundo boas práticas atuais, e configurar mapeamento automático de unidade de rede para a OU Financeiro, usando um grupo de segurança dedicado para controle de acesso.

## Ambiente

- **Domain Controller:** `srv-dc01` (lab.local)
- **Pasta compartilhada:** `C:\Compartilhado\Financeiro` em `srv-dc01`, exposta como `\\srv-dc01\Financeiro`
- **VM cliente de teste:** `USR-LAB-001`

## Etapas realizadas

### 1. Reforço da política de senha (Default Domain Policy)

Editada a **Default Domain Policy**, em `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`:

| Configuração | Valor anterior | Valor ajustado |
|---|---|---|
| Minimum password length | 7 caracteres | 12 caracteres |
| Maximum password age | 42 dias | 90 dias |

Demais configurações mantidas no padrão (Enforce password history: 24; Minimum password age: 1 dia; Complexity requirements: Enabled; Reversible encryption: Disabled).

**Justificativa:** alinhamento com diretrizes atuais (NIST), que priorizam senhas mais longas com expiração menos frequente, em vez de senhas curtas trocadas com alta frequência — abordagem que tende a gerar senhas previsíveis.

**Validação prática:** tentativa de alteração de senha do usuário `usuario.ti` para um valor abaixo de 12 caracteres, rejeitada pelo AD DS com a mensagem "The password does not meet the password policy requirements". Nova tentativa com senha de 12+ caracteres e complexidade adequada, aceita com sucesso.

### 2. Criação de grupo de segurança

Criado o grupo `GG-Financeiro` dentro da OU Financeiro:

- Group scope: **Global**
- Group type: **Security**
- Membro adicionado: `usuario.financeiro`

**Conceito reforçado:** distinção prática entre OU (localização estrutural do objeto, usada para aplicar GPOs) e grupo de segurança (usado para conceder permissões de acesso a recursos, como pastas compartilhadas). Um usuário pode estar em uma OU sem pertencer ao grupo de segurança correspondente — cenário útil para aplicar restrições de GPO a todos os membros de um departamento, mas reservar acesso a recursos sensíveis apenas a um subconjunto (ex.: durante período de experiência).

### 3. Criação de pasta compartilhada e permissões

Criada a pasta `C:\Compartilhado\Financeiro` em `srv-dc01`, compartilhada como `Financeiro`, com permissões de leitura e gravação concedidas ao grupo `GG-Financeiro` (em vez de conceder acesso genérico a "Todos").

**Observação de boas práticas:** hospedar compartilhamentos de arquivos diretamente em um Domain Controller não é a prática recomendada em ambientes de produção (o ideal seria um File Server dedicado) — decisão adotada aqui apenas para fins de aprendizado no laboratório.

### 4. Mapeamento automático de unidade via GPO Preferences

Criada uma nova GPO, `GPO-Financeiro-MapeamentoRede`, vinculada à OU Financeiro, configurada em `User Configuration → Preferences → Windows Settings → Drive Maps`:

- Action: **Create**
- Location: `\\srv-dc01\Financeiro`
- Drive Letter: **Z:**
- Label: `Financeiro`
- Reconnect: habilitado

**Problema encontrado:** configuração inicial usando Action "Update" e letra de unidade padrão (D:) não produzia o resultado esperado — "Update" pressupõe um mapeamento pré-existente para ajustar, e não cria um mapeamento novo do zero.

**Correção:** ajuste da Action para "Create" e da letra de unidade para "Z:", conforme convenção comum para unidades de rede.

**Validação prática:** login na máquina cliente com o usuário `usuario.financeiro`, execução de `gpupdate /force`, e confirmação de que a unidade `Financeiro (Z:)` passou a aparecer automaticamente em "Este Computador", com acesso de leitura/gravação funcionando (herdado da permissão concedida ao grupo `GG-Financeiro`).

## Conceitos aplicados

- Diretrizes atuais de política de senha (comprimento vs. frequência de troca) e o raciocínio de segurança por trás de cada configuração de Account Policy.
- Diferença funcional entre OU e grupo de segurança, e cenários onde essa distinção é usada deliberadamente (GPO aplicada a todos da OU, acesso a recursos restrito a um subconjunto via grupo).
- Group Policy Preferences (Drive Maps) como mecanismo distinto das políticas "clássicas" usadas em módulos anteriores — com semânticas próprias de Action (Create/Update/Replace/Delete).
- Importância de a Action refletir a intenção real (criar vs. atualizar um recurso já existente).

## Próximos passos

- Explorar scripts de logon (Logon Scripts) como alternativa/complemento ao Drive Maps.
- Considerar migração futura de compartilhamentos de arquivo para um File Server dedicado, separado dos Domain Controllers.

## Screenshots

_Ver pasta [`screenshots/`](./screenshots/)._
