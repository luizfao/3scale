# Red Hat 3scale API Management 2.16 via OpenShift GitOps (Argo CD)

Este repositório contém manifests **GitOps** para instalar Red Hat 3scale API Management **2.16** em um cluster OpenShift, em um namespace dedicado, com integração de SSO opcional ao **Keycloak (RHBK)** já implantado no namespace `rhbk-gitops` (repositório [keycloak](https://github.com/luizfao/keycloak)).

Documentação do produto: [Red Hat 3scale API Management 2.16](https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.16/)

## O que é implantado

- **Namespace**: `3scale-gitops`
- **Bancos externos (obrigatórios no 2.16)** no namespace **`3scale-databases`**, via GitOps:
  - **PostgreSQL** para System (`system_production`)
  - **Redis** para System (Sidekiq) e para Backend (storage + queues, DBs lógicos 0 e 1)
- **Zync database** em modo interno (gerenciado pelo operador), conforme suporte do 3scale 2.16
- **3scale Operator** (OLM) no canal `threescale-2.16`, aprovação de Install Plan **Manual**
- **APIManager** CR com `externalComponents` apontando para esses bancos

A integração com **SSO (RHBK/Keycloak)** é feita **depois** da instalação no 3scale, usando o Keycloak no namespace `rhbk-gitops` (repositório [keycloak](https://github.com/luizfao/keycloak)) como IdP para **Admin Portal**, **Developer Portal** e **APIs** (OIDC).

## Como usar (rápido)

### Pré-requisitos

- Cluster OpenShift com permissões de administrador
- **OpenShift GitOps (Argo CD)** instalado (namespace `openshift-gitops`)
- (Opcional) Keycloak (RHBK) já rodando em `rhbk-gitops` para SSO — use o repositório [keycloak](https://github.com/luizfao/keycloak) como referência

### 1) Definir o domínio do cluster (wildcardDomain)

O `APIManager` usa `spec.wildcardDomain`, que deve ser o domínio base das rotas do OpenShift (ex.: `apps.cluster-xxx.dynamic.redhatworkshops.io`). Obtenha o domínio do ingress:

```bash
oc get ingresscontroller -n openshift-ingress-operator -o jsonpath='{.items[0].status.domain}'
```

Edite `gitops/apimanager/apimanager.yaml` e substitua `spec.wildcardDomain: apps.example.com` pelo valor obtido.

### 2) (Se o repositório for privado) Credenciais no Argo CD

Crie um token de leitura no GitHub (Fine-grained PAT recomendado) com acesso a **Contents** e **Metadata** deste repositório. Depois aplique o Secret no cluster:

```bash
export GITHUB_TOKEN='<SEU_TOKEN>'

oc -n openshift-gitops apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: repo-3scale
  namespace: openshift-gitops
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com/USER/3scale.git
  username: x-access-token
  password: ${GITHUB_TOKEN}
EOF
```

Ou use o exemplo em `bootstrap/repo-credentials-secret.example.yaml` (copie para `repo-credentials-secret.yaml`, preencha o token; o arquivo está no `.gitignore`).

### 3) Ajustar a URL do repositório no Application

Edite `bootstrap/application.yaml` e substitua `repoURL: https://github.com/USER/3scale.git` pela URL real do seu repositório.

### 4) Bootstrap do Argo CD (dois Applications, sem editar arquivos)

São usados **dois** Applications: um para namespaces, operator, bancos e secrets; outro só para o APIManager (que depende do CRD criado pelo operador). Assim não é preciso alterar nenhum arquivo entre o primeiro e o segundo sync.

Aplique os dois Applications de uma vez:

```bash
oc apply -f bootstrap/application.yaml
oc apply -f bootstrap/application-apimanager.yaml
```

- **Application `3scale`**: sincronize no Argo CD. Cria namespaces `3scale-gitops` e `3scale-databases`, operator (Subscription/OperatorGroup), bancos externos (PostgreSQL System + Redis) e secrets. O APIManager fica de fora (excluído neste Application).
- **Application `3scale-apimanager`**: pode ficar em estado de falha até o CRD existir. **Não é preciso editá-lo** — depois que o operador estiver instalado (passo 5), basta sincronizar este Application no Argo CD.

### 5) Aprovar o Install Plan do operador (Manual) — depois sincronize o Application `3scale-apimanager`

O CRD `APIManager` só existe depois que o operador 3scale (CSV) está instalado. **Depois do sync do Application `3scale`** (namespaces, operator, DBs), aprove o Install Plan. Quando o CSV estiver **Succeeded**, sincronize o Application **`3scale-apimanager`** no Argo CD para aplicar o APIManager (não é preciso alterar nenhum arquivo).

Liste e aprove o Install Plan:

```bash
oc -n 3scale-gitops get installplans
# Aprove o Install Plan (substitua <nome> pelo nome retornado acima):
oc -n 3scale-gitops patch installplan <nome> -p '{"spec":{"approved":true}}' --type=merge
# Aguarde o operador ficar instalado (CSV Succeeded):
oc -n 3scale-gitops get csv -w
# Quando o CSV estiver Succeeded, sincronize o Application 3scale-apimanager no Argo CD
```

### 6) URLs após a implantação

Com `wildcardDomain` configurado (ex.: `apps.cluster-xxx.dynamic.redhatworkshops.io`):

- **Admin Portal (tenant padrão)**:
  - `https://3scale-admin.<wildcardDomain>`
- **Master Admin Portal** (criar outros tenants):
  - `https://master.<wildcardDomain>`

Exemplo: se `wildcardDomain` for `apps.cluster-zrdcz.dynamic.redhatworkshops.io`:
- Admin: `https://3scale-admin.apps.cluster-zrdcz.dynamic.redhatworkshops.io`
- Master: `https://master.apps.cluster-zrdcz.dynamic.redhatworkshops.io`

### 7) Credenciais do Admin e Master Portal

Obtenha do Secret `system-seed` no namespace `3scale-gitops`:

**Admin Portal:**

```bash
oc -n 3scale-gitops get secret system-seed -o jsonpath='{.data.ADMIN_USER}' | base64 -d; echo
oc -n 3scale-gitops get secret system-seed -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d; echo
```

**Master Admin Portal:**

```bash
oc -n 3scale-gitops get secret system-seed -o jsonpath='{.data.MASTER_USER}' | base64 -d; echo
oc -n 3scale-gitops get secret system-seed -o jsonpath='{.data.MASTER_PASSWORD}' | base64 -d; echo
```

## Integração SSO com RHBK (Keycloak em rhbk-gitops)

O 3scale usa o **Red Hat Build of Keycloak (RHBK)** já implantado no namespace `rhbk-gitops` (repositório [keycloak](https://github.com/luizfao/keycloak)) como provedor de identidade único para **Admin Portal**, **Developer Portal** e **APIs**. A configuração é feita no Admin Portal do 3scale e no console do Keycloak; não há parâmetros de SSO no APIManager CR.

**Pré-requisito:** Keycloak (RHBK) rodando em `rhbk-gitops` com um realm (ex.: `rhbk`) e a URL do realm acessível (ex.: `https://rhbk-rhbk-gitops.<cluster-domain>/realms/rhbk`).

---

### 1) SSO para Admin Portal (membros e administradores)

Permite que usuários e administradores do 3scale façam login com credenciais do Keycloak.

- **No Keycloak (realm usado pelo 3scale):**
  - Crie um **client** para o 3scale Admin Portal (tipo: OpenID Connect, acesso público ou confidencial).
  - Configure **Redirect URIs** com o callback do 3scale (veja passo abaixo).
  - Anote **Client ID** e **Client Secret**.

- **No 3scale Admin Portal:**
  - **Account Settings** (ícone de engrenagem) → **Users** → **SSO Integrations** → **Create a new SSO integration**.
  - Informe:
    - **Realm or Site**: URL do realm do Keycloak (ex.: `https://rhbk-rhbk-gitops.<cluster-domain>/realms/rhbk`).
    - **Client**: Client ID do Keycloak.
    - **Client Secret**: secret do client.
  - O 3scale exibe a **Callback URL** (ex.: `https://3scale-admin.<wildcardDomain>/auth/<system_name>/callback`). Use essa URL em **Valid Redirect URIs** do client no Keycloak.

Referência: [Admin Portal – Red Hat single sign-on and Red Hat build of Keycloak](https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.15/html/admin_portal_guide/admin-portal-sso).

---

### 2) SSO para Developer Portal (desenvolvedores)

Permite que desenvolvedores que acessam o Developer Portal façam login via Keycloak.

- **No Keycloak:** use o mesmo realm (ou outro) e crie um client para o Developer Portal, com redirect URIs apontando para o Developer Portal do 3scale.

- **No 3scale Admin Portal:**
  - **Audience** → **Developer Portal** → **SSO Integrations**.
  - Crie uma integração SSO informando o **Realm/Site** do Keycloak, **Client ID** e **Client Secret**, e configure a **Callback URL** no client do Keycloak conforme indicado pelo 3scale.

Referência: [Creating the Developer Portal – Authentication](https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.15/html/creating_the_developer_portal/authentication).

---

### 3) SSO/OIDC para APIs (segurança dos produtos)

Permite que as APIs gerenciadas pelo 3scale exijam tokens JWT emitidos pelo Keycloak (OIDC).

- **No Keycloak (realm, ex.: `rhbk`):**
  - Crie um **client** para o **Zync** (3scale) com permissões de **Realm Management** (ex.: `manage-clients`), para o 3scale sincronizar aplicações e credenciais.
  - Crie clients/issuers conforme necessário para os consumidores das APIs.

- **No 3scale Admin Portal (por produto):**
  - **Integration** → **Configuration** → **Authentication**.
  - Selecione **OpenID Connect**, tipo **Red Hat Single Sign-On**.
  - Configure:
    - **OpenID Connect Issuer**: URL do realm (ex.: `https://rhbk-rhbk-gitops.<cluster-domain>/realms/rhbk`).
    - **Zync client**: Client ID e secret do client Zync no Keycloak.
  - Após alterar o modo de autenticação, pode ser necessário **recriar aplicações** para o Zync sincronizar credenciais com o Keycloak.

Referência: [Securing APIs using OIDC with Red Hat Single Sign-On](https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.16/html-single/admin_portal_guide/) (Admin Portal / Authentication).

## Estrutura do repositório (manifestos)

| Caminho | Descrição |
|--------|------------|
| `bootstrap/application.yaml` | Argo CD Application `3scale` (monitora `gitops/`, exclui `apimanager/`) |
| `bootstrap/application-apimanager.yaml` | Argo CD Application `3scale-apimanager` (monitora só `gitops/apimanager/`) |
| `bootstrap/repo-credentials-secret.example.yaml` | Exemplo de Secret para repositório privado |
| `gitops/namespace.yaml` | Namespaces `3scale-gitops` e `3scale-databases` |
| `gitops/operator/operatorgroup.yaml` | OperatorGroup (operador no namespace) |
| `gitops/operator/subscription.yaml` | Subscription 3scale Operator 2.16 (Manual) |
| `gitops/databases/postgresql-system/postgresql.yaml` | PostgreSQL para System (`system_production`) em `3scale-databases` |
| `gitops/databases/redis-system/redis.yaml` | Redis para System (Sidekiq) em `3scale-databases` |
| `gitops/databases/redis-backend/redis.yaml` | Redis para Backend (storage + queues) em `3scale-databases` |
| `gitops/databases/3scale-secrets.yaml` | Secrets `system-database`, `system-redis`, `backend-redis` para o operador |
| `gitops/apimanager/apimanager.yaml` | APIManager CR (`wildcardDomain` + `externalComponents`) |

## Observações

- **2.16 e bancos externos**: a partir do 2.16, system database, system Redis e backend Redis são obrigatórios como componentes externos. Este repositório instala PostgreSQL (system) e Redis (system + backend) no namespace **`3scale-databases`** via GitOps; os secrets em `3scale-gitops` referenciam os serviços por FQDN (`*.3scale-databases.svc.cluster.local`). O APIManager usa `externalComponents` para referenciar esses secrets.
- **Zync database**: mantido em modo interno (gerenciado automaticamente pelo operador), conforme [suportado no 3scale 2.16](https://access.redhat.com/articles/2798521).
- **Senhas**: os secrets em `gitops/databases/` usam placeholders (`change-me`). Mantenha o `system-database` consistente com o secret `postgresql-system`.
- **Persistent volumes**: os StatefulSets de PostgreSQL e Redis usam PVCs RWO; o operador 3scale continua responsável por volumes RWX do portal quando em modo externo.
- **Canal do operador**: `threescale-2.16`. Confirme no OperatorHub do cluster o nome do pacote e do canal se houver diferença.
- **Repositório Keycloak**: para implantar ou ajustar o RHBK usado no SSO (Admin, Developer Portal e APIs), use o repositório [keycloak](https://github.com/luizfao/keycloak) e o namespace `rhbk-gitops`.
