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

Ver [SURVEYS.md](SURVEYS.md) para o detalhe dos surveys dos Job Templates.

## Requisitos

- AAP 2.7 com collection `ansible.controller >=4.8.0`.
- Credential do tipo "Red Hat Ansible Automation Platform" anexada aos Job
  Templates (injeta `CONTROLLER_HOST`/`CONTROLLER_USERNAME`/
  `CONTROLLER_PASSWORD` ou `CONTROLLER_OAUTH_TOKEN`).

## Como usar no seu AAP

Nada no código tem nome de Organization/Team fixo — `Infra`, `Windows`,
`Linux` são só exemplos digitados no survey na hora de rodar. O que precisa
ser configurado é a plataforma em si:

### 1. No AAP (sem mexer em código)

Crie, apontando pro **seu** ambiente:

| Item | O que configurar |
|---|---|
| Execution Environment | confirme que tem a collection `ansible.controller >=4.8.0` (se não tiver, construa uma EE customizada com ela) |
| Project | aponte a SCM URL para o seu fork/clone deste repositório (`https://github.com/<seu-usuario>/AAP-RBAC.git`) |
| Credential | crie uma do tipo **Red Hat Ansible Automation Platform**, com host/usuário/senha (ou token) do **seu** controller |
| Inventory | um Inventory simples com `localhost` (`ansible_connection=local`) — os playbooks só chamam API, não tocam em host nenhum |
| Job Templates | crie os dois (`RBAC - Criar Organização com Teams` e `RBAC - Definir Perfil de Usuário`), ligados ao Project/Credential/Inventory acima, com os surveys descritos em [SURVEYS.md](SURVEYS.md) |

### 2. No código (opcional, só se quiser mudar o padrão)

| Arquivo | O que dá pra ajustar |
|---|---|
| `vars/rbac/personas.yml` | a lista dos 4 perfis (`Administrator, Operator, Developer, Auditor`). Mudar aqui exige também ajustar `vars/rbac/roles.yml` com o modelo de permissão do perfil novo/renomeado |
| `vars/rbac/roles.yml` | as permissões de cada perfil (ex.: dar `delete` para o Operator, tirar `execute` do Auditor, adicionar outro tipo de recurso) |
| `roles/rbac/meta/main.yml` | troque `author: "SEU_NOME_AQUI"` pelo seu nome/time |
| `collections/requirements.yml` | só se precisar fixar outra versão mínima da collection `ansible.controller` |

Na maioria dos casos **não precisa mexer em nenhum desses arquivos** — o
survey já cobre organização, áreas, usuário e perfil na hora de rodar.
