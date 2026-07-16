# Módulo 01 — Setup do Primeiro Domain Controller (DC01)

## Objetivo

Criar e promover o primeiro Domain Controller do laboratório de infraestrutura, estabelecendo a floresta/domínio `lab.local`.

## Ambiente

- **Hypervisor:** Proxmox VE 9.2.2
- **VM:** `vm-dc01-lab` (hostname `srv-dc01`)
- **Specs:** 2 vCPU, 4GB RAM, 60GB disco (SCSI, controller VirtIO SCSI single)
- **BIOS:** OVMF (UEFI) + TPM 2.0 — requisito do Windows Server 2025
- **SO:** Windows Server 2025 Standard Evaluation (Desktop Experience)
- **IP estático:** 192.168.100.201

## Etapas realizadas

### 1. Criação da VM no Proxmox

VM criada com BIOS UEFI, disco em barramento SCSI (VirtIO SCSI single) e Guest OS configurado como "Microsoft Windows 11/2022/2025" (agrupamento correto no Proxmox 9.x, já que Windows Server 2025 compartilha o mesmo kernel base do Windows 11).

### 2. Instalação do Windows Server 2025

**Problema encontrado:** o instalador não reconhecia o disco rígido na etapa de particionamento ("Select location to install Windows Server" sem discos listados).

**Causa:** o controller de disco VirtIO SCSI não é reconhecido nativamente pelo instalador do Windows — exige driver de terceiros.

**Solução:**
- Download da ISO oficial de drivers VirtIO: [virtio-win.iso](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso)
- ISO montada como segundo CD-ROM (barramento IDE) na VM
- Driver carregado via "Load Driver" no instalador → pasta `vioscsi` → versão correspondente → `amd64`
- Disco reconhecido normalmente após o carregamento do driver

**Segundo problema:** primeira instalação concluída acidentalmente na edição **sem Desktop Experience** (Server Core), diferente do padrão usado no ambiente de trabalho. Reinstalação necessária, escolhendo a edição "Windows Server 2025 Standard (Desktop Experience)".

**Ajuste de boot order:** para evitar depender do timing manual da tela "Press any key to boot from CD or DVD", a ordem de boot foi ajustada diretamente em **Options → Boot Order**, priorizando o CD-ROM — solução mais confiável que a tentativa manual.

### 3. Configuração inicial

- Hostname alterado para `srv-dc01`
- IP estático configurado: `192.168.100.201`
- Atualizações do Windows aplicadas

### 4. Instalação do papel AD DS

Papel **Active Directory Domain Services** instalado via Server Manager (Add Roles and Features), incluindo as ferramentas de gerenciamento (RSAT).

### 5. Promoção a Domain Controller

Servidor promovido via **Active Directory Domain Services Configuration Wizard**:

- Tipo de implantação: **Add a new forest**
- Domínio raiz: `lab.local`
- Forest/Domain functional level: **Windows Server 2025**
- DNS Server: habilitado (co-localizado no DC)
- Global Catalog: habilitado (obrigatório para o primeiro DC da floresta)
- RODC: desabilitado
- Senha DSRM configurada separadamente da senha de Administrador do domínio

### 6. Validação pós-promoção

Comandos executados via PowerShell para confirmar o funcionamento:

```powershell
Get-ADDomain
Get-ADDomainController
Resolve-DnsName lab.local
```

**Resultado:** domínio `lab.local` ativo; `srv-dc01` reconhecido como Domain Controller, Global Catalog (`IsGlobalCatalog: True`), detentor de todas as 5 FSMO roles (`SchemaMaster`, `DomainNamingMaster`, `PDCEmulator`, `RIDMaster`, `InfrastructureMaster`); resolução DNS confirmada apontando corretamente para `192.168.100.201`.

## Conceitos aplicados

- Diferença entre conta de Administrador **local** (base SAM, isolada por máquina) e conta de Administrador **de domínio** (base NTDS.dit, compartilhada e replicada entre DCs) — e a implicação de segurança de comprometer essa conta.
- Papéis FSMO (Flexible Single Master Operations) e por que todos ficam no primeiro DC de uma floresta nova.
- Dependência estrutural entre AD e DNS.

## Próximos passos

- Criação do DC02 e configuração de replicação entre controladores de domínio.

## Screenshots

_Ver pasta [`screenshots/`](./screenshots/)._
