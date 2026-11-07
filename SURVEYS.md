# Surveys — RBAC AAP 2.7

Job Templates a criar no AAP para `playbooks/rbac_org_create.yml` e
`playbooks/rbac_assign_user.yml` (role `rbac`, API do gateway) e as
perguntas de cada survey.

## Modelo

- **Organizations** (ex.: `Infra`), cada uma com seus **Teams** (ex.:
  `Windows`, `Linux`).
- **Cada Team tem 4 roles prontas**: `{Team} - Administrator`,
  `{Team} - Operator`, `{Team} - Developer`, `{Team} - Auditor` (criadas
  pelo JT "RBAC - Criar Organização com Teams").
- **O perfil de cada pessoa** é a role do Team dela, vinculada **direto no
  usuário** (nunca no Team) pelo JT "RBAC - Definir Perfil de Usuário".
- As roles são de Organization: valem para todos os objetos da Organization
  do Team. Para isolar áreas, use Organizations diferentes.
- Isso vale pra **qualquer** Organization/Team/usuário — não há nada fixo
  no código.

| JT (sugerido) | Nome | Playbook | Survey |
|---|---|---|---|
| RBAC - Criar Organização com Teams | `playbooks/rbac_org_create.yml` | Sim |
| RBAC - Definir Perfil de Usuário | `playbooks/rbac_assign_user.yml` | Sim |

Credential (ambos): `AAP Controller - Admin` (tipo *Red Hat Ansible
Automation Platform*) · Inventory: `Localhost` ·
EE: `Default execution environment` · Project: este repositório (`AAP-RBAC`)

---

## JT — RBAC - Criar Organização com Teams

Cria uma Organization e, para cada área informada, um Team simples (só
agrupamento, sem role). Organizations e Teams que já existem **não são
recriados nem alterados**.

### Perguntas

| # | Pergunta | Variável | Tipo | Obrigatória | Padrão |
|---|---|---|---|---|---|
| 1 | Nome da Organization | `rbac_org_name` | Texto (1–512) | Sim | — |
| 2 | Descrição da Organization | `rbac_org_description` | Texto (0–1024) | Não | — |
| 3 | Áreas (uma por linha) | `rbac_org_teams` | Área de texto (1–4096) | Sim | `Windows`<br>`Linux` |

### Criar o survey no JT

Pela UI: Templates → `RBAC - Criar Organização com Teams` → aba **Survey**
→ **Add** → crie as 3 perguntas da tabela acima, na ordem, com os mesmos
tipos/variáveis/limites.

---

## JT — RBAC - Definir Perfil de Usuário

Vincula ao usuário (direto nele, nunca no Team) a role `{Team} - {Perfil}`.
Ex.: `jsilva`, Team `Linux`, perfil `Auditor` → recebe `Linux - Auditor`.

O que o job faz:

1. garante as 4 roles do Team (`{Team} - Administrator/Operator/Developer/Auditor`);
2. vincula ao usuário a role `{Team} - {Perfil}` na Organization do Team;
3. remove as outras roles desse Team que ele tinha (ex.: era Operator, virou
   Auditor) — ele fica **só** com o perfil escolhido. Roles de outros Teams
   não são tocadas.

> O usuário **não** é adicionado como membro (`Team Member`) do Team: o
> Team só serve para nomear a role. A permissão vem exclusivamente da role
> `{Team} - {Perfil}`.

> As roles são de **Organization**: valem para **todos** os objetos da
> Organization do Team (inclusive os criados depois). Um `Linux - Auditor`
> numa Org que também tem o Team `Windows` vê os objetos do Windows também.
> Para isolar áreas, use Organizations diferentes.

### Perguntas

| # | Pergunta | Variável | Tipo | Obrigatória | Padrão |
|---|---|---|---|---|---|
| 1 | Username | `rbac_user_username` | Texto (1–150) | Sim | — |
| 2 | Organization | `rbac_org_name` | Texto (1–512) | Sim | `Infra` |
| 3 | Team | `rbac_team_name` | Texto (1–512) | Sim | — |
| 4 | Perfil | `rbac_persona` | Múltipla escolha: Administrator / Operator / Developer / Auditor | Sim | — |

- O usuário precisa **já existir** no AAP (usuário do AD só aparece depois
  do primeiro login).
- O Team precisa existir (JT "RBAC - Criar Organização com Teams").

### As 4 roles de cada Team

Modelos em `vars/rbac/roles.yml` (`Custom - {Perfil}`, nunca criados com esse
nome — viram `{Team} - {Perfil}`).

| Perfil | Credential | Inventory | Project | Job Template | Workflow |
|---|---|---|---|---|---|
| Administrator | full | full | full | full | full |
| Operator | read·update·execute | read·update·execute | read·update·execute | read·update·execute | read·update·execute |
| Developer | read·execute | read·execute | read·update·execute | read·update·execute | read·update·execute |
| Auditor | read | read | read | read | read |

Administrator = mesmas permissões das roles nativas "Organization {Recurso}
Admin" (inclui criar e apagar).
