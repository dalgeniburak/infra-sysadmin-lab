# Módulo 02 — DC02 e Replicação entre Controladores de Domínio

## Objetivo

Adicionar um segundo Domain Controller ao domínio `lab.local`, eliminando o ponto único de falha (single point of failure) e validando a replicação do Active Directory.

## Ambiente

- **VM:** `vm-dc02-lab` (hostname `srv-dc02`)
- **Specs:** 2 vCPU, 4GB RAM, 60GB disco (SCSI, VirtIO SCSI single)
- **SO:** Windows Server 2025 Standard Evaluation (Desktop Experience)
- **IP estático:** 192.168.100.202
- **DNS preferencial:** 192.168.100.201 (apontando para o DC01)

## Etapas realizadas

### 1. Configuração inicial

Hostname alterado para `srv-dc02`, IP estático configurado, e DNS preferencial apontado explicitamente para o DC01 (`192.168.100.201`) — passo crítico, já que a entrada no domínio depende dos registros SRV armazenados no DNS do DC01.

### 2. Entrada no domínio como membro

**Problema encontrado:** primeira tentativa de entrar no domínio (`System Properties → Change → Domain`) falhou com o erro:

> "An Active Directory Domain Controller (AD DC) for the domain 'lab.local' could not be contacted."

**Diagnóstico:** verificação em camadas via PowerShell:
```powershell
Get-NetIPConfiguration
Test-Connection 192.168.100.201
Resolve-DnsName lab.local
nslookup lab.local 192.168.100.201
```

**Causa raiz:** configuração de DNS incompleta/incorreta no adaptador de rede do DC02 — o servidor não conseguia resolver os registros SRV do domínio, mesmo com conectividade de rede (ping) funcionando. Isso reforça a diferença entre "alcançar via rede" e "ter o registro DNS correto": um DNS genérico (ex.: 8.8.8.8) nunca teria os registros do domínio interno `lab.local`, por não ser autoritativo para essa zona.

**Correção:** ajuste do DNS preferencial no adaptador de rede, apontando corretamente para 192.168.100.201.

Após a correção, entrada no domínio bem-sucedida.

### 3. Instalação do papel AD DS

Papel **Active Directory Domain Services** instalado via Server Manager, mesmo processo do DC01.

### 4. Promoção a Domain Controller adicional

Via **Active Directory Domain Services Configuration Wizard**:

- Tipo de implantação: **Add a domain controller to an existing domain**
- Domain: `lab.local`

**Problema encontrado:** erro "You must supply a user account name" — o assistente estava usando a conta de Administrador **local** do DC02 (`SRV-DC02\Administrator`), sem permissão para promover um DC no domínio.

**Correção:** troca explícita das credenciais via botão "Change...", utilizando a conta de Administrador do domínio (`LAB\Administrator`).

**Aviso esperado (não é erro):** na etapa de DNS Options, o assistente exibiu o aviso "A delegation for this DNS server cannot be created because the authoritative parent zone cannot be found" — esperado em domínios isolados como `lab.local`, que não possuem domínio pai na internet pública. Opção "Update DNS delegation" mantida desmarcada, sem impacto na instalação.

Instalação concluída e servidor reiniciado.

### 5. Validação da replicação

Comandos executados para confirmar o funcionamento:

```powershell
Get-ADDomainController -Filter *
Get-ADReplicationPartnerMetadata -Target srv-dc02.lab.local
```

**Teste prático de replicação:** criação de um usuário de teste em um DC e confirmação de sua presença no outro:
```powershell
New-ADUser -Name "teste-replicacao" -Enabled $false
Get-ADUser -Filter {Name -eq "teste-replicacao"}
```

**Resultado:** ambos os controladores (`srv-dc01` e `srv-dc02`) reconhecidos como DCs do domínio; replicação confirmada em ambas as direções.

## Conceitos aplicados

- **Redundância / alta disponibilidade:** um único DC representa um ponto único de falha — se cair, autenticação, DNS interno e acesso a recursos do domínio param para toda a rede.
- **Dependência de DNS na entrada de domínio:** o Windows localiza controladores de domínio através de registros SRV, armazenados apenas no(s) DNS que hospeda(m) a zona do domínio — não em DNS públicos.
- **Modelo multi-master do AD** e o papel das **FSMO roles** (Schema Master, Domain Naming Master, PDC Emulator, RID Master, Infrastructure Master) — operações sensíveis a conflito que exigem um único "dono" por vez, mesmo em um modelo onde qualquer DC aceita escrita.
- **Replicação:** notificação + intervalo (padrão ~15s no mesmo site), replicação urgente para mudanças críticas (ex.: reset de senha), e uso de `Get-ADReplicationPartnerMetadata` para diagnóstico de saúde de replicação.

