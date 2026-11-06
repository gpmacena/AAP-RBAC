# AAP-RBAC

Automação de RBAC para o Ansible Automation Platform 2.7, via API do
gateway (Organizations, Teams e Custom Roles).

## Estrutura

```
.
├── ansible.cfg
├── collections/requirements.yml      # ansible.controller >=4.8.0
├── playbooks/
│   ├── rbac_org_create.yml           # cria Organization + Teams de área + 4 roles por Team
│   └── rbac_assign_user.yml          # vincula um usuário a uma das 4 roles de um Team
├── roles/rbac/
│   ├── tasks/
│   │   ├── org_create.yml
│   │   ├── area_roles.yml            # gera/garante as 4 roles "{Team} - {Perfil}"
│   │   ├── ensure_roles.yml          # cria/corrige Custom Roles via API (controller + gateway)
│   │   └── assign_user.yml
│   ├── defaults/main.yml
│   └── meta/main.yml
└── vars/rbac/
    ├── roles.yml                     # catálogo de permissões por perfil (modelo)
    └── personas.yml                  # lista dos 4 perfis
```

## Modelo

- 1 Organization por contexto (ex.: `Infra`).
- 1 Team por área dentro dela (ex.: `Windows`, `Linux`) — só agrupamento,
  sem permissão.
- Cada Team tem 4 Custom Roles prontas: `{Team} - Administrator`,
  `{Team} - Operator`, `{Team} - Developer`, `{Team} - Auditor`.
- O perfil de cada pessoa é atribuído **direto nela** (usuário), nunca no
  Team — permite perfis diferentes dentro do mesmo Team.
- As roles são de Organization: valem para todos os objetos dela. Para
  isolar áreas de verdade, use Organizations diferentes.

Ver [SURVEYS.md](SURVEYS.md) para o detalhe dos surveys dos Job Templates e
[AAP_RBAC_SETUP_PROMPT.md](AAP_RBAC_SETUP_PROMPT.md) para o passo a passo de
configuração no AAP (Project, Credential, Inventory, Job Templates).

## Requisitos

- AAP 2.7 com collection `ansible.controller >=4.8.0`.
- Credential do tipo "Red Hat Ansible Automation Platform" anexada aos Job
  Templates (injeta `CONTROLLER_HOST`/`CONTROLLER_USERNAME`/
  `CONTROLLER_PASSWORD` ou `CONTROLLER_OAUTH_TOKEN`).
