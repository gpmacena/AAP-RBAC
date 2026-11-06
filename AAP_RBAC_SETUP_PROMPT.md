# Prompt — Criar estrutura RBAC no AAP 2.7

Execute os passos abaixo, na ordem, no Ansible Automation Platform 2.7. Se um
objeto já existir com o mesmo nome, apenas confirme a configuração (não
duplicar).

Organization de todos os objetos abaixo: **Default**.

## Modelo

- **1 Organization** (ex.: `Infra`) para todas as áreas.
- **1 Team por área** (ex.: `Windows`, `Linux`) — **só agrupamento**, sem
  role nenhuma vinculada.
- **O perfil de cada pessoa** (`Administrator`/`Operator`/`Developer`/
  `Auditor`) é atribuído **direto nela** (usuário), nunca no Team.

## 1. Project

Criar (ou verificar) o Project:

| Campo | Valor |
|---|---|
| Nome | `AAP-RBAC` |
| Organization | `Default` |
| Control de origem (SCM Type) | `Git` |
| SCM URL | URL deste repositório, depois de criado (ex.: `https://github.com/<voce>/AAP-RBAC.git`) |
| SCM Branch | `main` |
| SCM Credential | `Git SCM` (se o repositório for privado; se for público, deixar em branco) |
| Atualizar na execução (SCM Update on Launch) | `true` |

## 2. Credential

Criar (ou verificar) a credential usada pelos Job Templates de RBAC para
autenticar na própria API do AAP:

| Campo | Valor |
|---|---|
| Nome | `AAP Controller - Admin` |
| Organization | `Default` |
| Tipo de Credential | `Red Hat Ansible Automation Platform` |
| Host | URL do próprio controller (ex.: `https://aap.lab.local`) |
| Username / Password **ou** OAuth Token | credenciais de uma conta com permissão para gerenciar Organizations/Teams/Users/Role Definitions via API do gateway |
| Verify SSL | conforme o ambiente |

> Essa credential injeta as variáveis de ambiente `CONTROLLER_HOST`,
> `CONTROLLER_USERNAME`, `CONTROLLER_PASSWORD` (ou `CONTROLLER_OAUTH_TOKEN`)
> e `CONTROLLER_VERIFY_SSL`, usadas pela role `rbac` do projeto.

## 3. Inventory

Criar (ou verificar) o Inventory:

| Campo | Valor |
|---|---|
| Nome | `Localhost` |
| Organization | `Default` |
| Host | `localhost` com `ansible_connection=local` |

## 4. Execution Environment

Usar o `Default execution environment`. **Pré-requisito**: ele precisa ter a
collection `ansible.controller` na versão `>=4.8.0` (módulos
`role_definition`, `role_team_assignment` e `role_user_assignment` só
existem a partir dessa versão — ver `collections/requirements.yml` do
projeto). Se a EE padrão não tiver essa versão, construir/usar uma EE
customizada com essa collection antes de seguir.

## 5. Job Template — "RBAC - Criar Organização com Teams"

Cria uma Organization e, para cada área informada (ex.: `Windows`, `Linux`),
um Team simples (só agrupamento, sem role).

| Campo | Valor |
|---|---|
| Nome | `RBAC - Criar Organização com Teams` |
| Organization | `Default` |
| Project | `AAP-RBAC` |
| Playbook | `playbooks/rbac_org_create.yml` |
| Inventory | `Localhost` |
| Credentials | `AAP Controller - Admin` |
| Execution Environment | `Default execution environment` |
| Survey | **Ativado** (spec abaixo) |
| Ask Variables on Launch | Não |

### Survey spec

```bash
curl -sk -u admin:SENHA -H 'Content-Type: application/json' \
  -X POST https://aap.lab.local/api/controller/v2/job_templates/<ID>/survey_spec/ \
  -d '{
    "name": "",
    "description": "",
    "spec": [
      {"question_name": "Nome da Organization", "variable": "rbac_org_name", "type": "text", "min": 1, "max": 512, "required": true},
      {"question_name": "Descrição da Organization", "variable": "rbac_org_description", "type": "text", "min": 0, "max": 1024, "required": false},
      {"question_name": "Áreas (uma por linha)", "question_description": "Uma área por linha, ex.: Windows / Linux. Vira um Team simples por área.", "variable": "rbac_org_teams", "type": "textarea", "min": 1, "max": 4096, "required": true, "default": "Windows\nLinux"}
    ]
  }'
```

## 6. Job Template — "RBAC - Definir Perfil de Usuário"

Atribui, direto num usuário (já existente no AAP), as roles de Organization
do perfil escolhido. Não mexe em Team nenhum, a não ser que você informe um
para agrupamento.

| Campo | Valor |
|---|---|
| Nome | `RBAC - Definir Perfil de Usuário` |
| Organization | `Default` |
| Project | `AAP-RBAC` |
| Playbook | `playbooks/rbac_assign_user.yml` |
| Inventory | `Localhost` |
| Credentials | `AAP Controller - Admin` |
| Execution Environment | `Default execution environment` |
| Survey | **Ativado** (spec abaixo) |
| Ask Variables on Launch | Não |

### Survey spec

```bash
curl -sk -u admin:SENHA -H 'Content-Type: application/json' \
  -X POST https://aap.lab.local/api/controller/v2/job_templates/<ID>/survey_spec/ \
  -d '{
    "name": "",
    "description": "",
    "spec": [
      {"question_name": "Username", "variable": "rbac_user_username", "type": "text", "min": 1, "max": 150, "required": true},
      {"question_name": "Nome da Organization", "variable": "rbac_org_name", "type": "text", "min": 1, "max": 512, "required": true},
      {"question_name": "Perfil", "variable": "rbac_persona", "type": "multiplechoice", "choices": "Administrator\nOperator\nDeveloper\nAuditor", "required": true},
      {"question_name": "Team/área (opcional)", "variable": "rbac_team_name", "type": "text", "min": 0, "max": 512, "required": false}
    ]
  }'
```

## Validação pós-criação

1. Rodar `RBAC - Criar Organização com Teams` com Nome: `Infra`, Áreas:
   `Windows` / `Linux`. Confirmar que a Organization `Infra` existe com os
   Teams `Windows` e `Linux` dentro dela, e que cada Team já tem as 4
   roles prontas (`Windows - Administrator/Operator/Developer/Auditor`),
   mas **sem ninguém atribuído ainda**.
2. Rodar `RBAC - Definir Perfil de Usuário` com um usuário de teste,
   Organization `Infra`, Team `Windows`, Perfil `Operator`. Confirmar que o
   usuário recebeu a role `Windows - Operator` **nele mesmo** (não no
   Team), e que foi adicionado como membro do Team `Windows`.
3. Rodar o mesmo Job Template de novo, mudando o Perfil para `Auditor`.
   Confirmar que a role `Windows - Operator` foi removida do usuário e ele
   ficou só com `Windows - Auditor`.
