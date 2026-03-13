# Red Hat 3scale API Management 2.16 via OpenShift GitOps (Argo CD)
NOTA: Este repositório foi criado com ajuda de IA.

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

### Modos de implantação

Este repositório suporta dois modos:

- **1 cluster (simples):** todos os bootstrap files são aplicados no mesmo cluster, usando `3-application-apimanager.yaml`.
- **2 clusters / HA ativo-passivo:** os mesmos bootstrap files são aplicados nos dois clusters, exceto o APIManager — o cluster primário usa `3-application-apimanager.yaml` e o cluster HA usa `3-application-ha-apimanager.yaml` (que aponta para `gitops/apimanager-ha/` com o `wildcardDomain` do cluster HA).

### Pré-requisitos

- Cluster OpenShift com permissões de administrador
- **OpenShift GitOps (Argo CD)** instalado (namespace `openshift-gitops`)
- (Opcional) Keycloak (RHBK) já rodando em `rhbk-gitops` para SSO — use o repositório [keycloak](https://github.com/luizfao/keycloak) como referência

### 1) Definir o domínio do cluster (wildcardDomain)

O `APIManager` usa `spec.wildcardDomain`, que deve ser o domínio base das rotas do OpenShift (ex.: `apps.cluster-xxx.dynamic.redhatworkshops.io`). Obtenha o domínio do ingress:

```bash
oc get ingresscontroller -n openshift-ingress-operator -o jsonpath='{.items[0].status.domain}'
```

- Edite `gitops/apimanager/apimanager.yaml` e substitua `spec.wildcardDomain: apps.example.com` pelo valor obtido.
- Se usar modo 2 clusters, edite também `gitops/apimanager-ha/apimanager-ha.yaml` com o domínio do cluster HA.

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

Ou use o exemplo em `bootstrap/0-repo-credentials-secret.example.yaml` (copie para `0-repo-credentials-secret.yaml`, preencha o token; o arquivo está no `.gitignore`).

### 3) Ajustar a URL do repositório nos Applications

Edite todos os arquivos em `bootstrap/` e substitua `repoURL: https://github.com/luizfao/3scale.git` pela URL real do seu repositório.

### 4) Bootstrap do Argo CD

São usados **5 Applications**, aplicados em ordem numérica. O operador 3scale precisa estar instalado antes de aplicar o APIManager (passo 5 abaixo).

```bash
# Cluster PRIMÁRIO (e único, se modo 1 cluster):
oc apply -f bootstrap/1-application-operator.yaml
# Aguardar a instalação do operador (passo 5) antes de continuar
oc apply -f bootstrap/2-application-databases.yaml
oc apply -f bootstrap/3-application-apimanager.yaml        # cluster primário
oc apply -f bootstrap/4-application-apicast-selfmanaged.yaml
oc apply -f bootstrap/5-application-echoapi.yaml
```

```bash
# Cluster HA (modo 2 clusters) — mesmos passos, exceto o APIManager:
oc apply -f bootstrap/1-application-operator.yaml
# Aguardar a instalação do operador (passo 5) antes de continuar
oc apply -f bootstrap/2-application-databases.yaml
oc apply -f bootstrap/3-application-ha-apimanager.yaml     # cluster HA
oc apply -f bootstrap/4-application-apicast-selfmanaged.yaml
oc apply -f bootstrap/5-application-echoapi.yaml
```

Applications criados:

| Application Argo CD | Arquivo bootstrap | Diretório GitOps |
|---|---|---|
| `3scale-operator` | `1-application-operator.yaml` | `gitops/operator/` |
| `3scale-databases` | `2-application-databases.yaml` | `gitops/databases/` |
| `3scale-apimanager` | `3-application-apimanager.yaml` | `gitops/apimanager/` |
| `3scale-ha-apimanager` | `3-application-ha-apimanager.yaml` | `gitops/apimanager-ha/` |
| `3scale-apicast-selfmanaged` | `4-application-apicast-selfmanaged.yaml` | `gitops/apicast-selfmanaged/` |
| `3scale-echoapi` | `5-application-echoapi.yaml` | `gitops/echoapi/` |

### 5) Aprovar o Install Plan do operador (Manual) — depois sincronize `3scale-apimanager`

O CRD `APIManager` só existe depois que o operador 3scale (CSV) está instalado. **Depois do sync do Application `3scale-operator`**, aprove o Install Plan. Quando o CSV estiver **Succeeded**, sincronize o Application **`3scale-apimanager`** (ou `3scale-ha-apimanager` no cluster HA) no Argo CD.

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

### 8) Testar as APIs (APIcast Self-Managed)

O APIcast Self-Managed usa hostnames customizados que precisam de `--resolve` para testes locais (não há DNS real apontando para eles). Obtenha o IP do router a partir da Route do console (presente em qualquer cluster OpenShift):

```bash
ROUTER_IP=$(nslookup \
  "$(oc get route console -n openshift-console -o jsonpath='{.spec.host}')" \
  | awk '/^Address/ && !/#/ {print $2}')
echo "Router IP: ${ROUTER_IP}"
```

Hostnames dos produtos:

| Produto | Staging | Production |
|---------|---------|-----------|
| echoapi (user_key) | `api-staging.example.com` | `api.example.com` |
| echoapi-oidc | `api-oidc-staging.example.com` | `api-oidc.example.com` |
| echoapi-rhbk-auth | `api-rhbk-staging.example.com` | `api-rhbk.example.com` |

Para os comandos de teste completos de cada produto, consulte:
- `ECHOAPI_MANUAL_STEPS.md` — Echo API (user_key)
- `ECHOAPI_OIDC_MANUAL_STEPS.md` — Echo API OIDC
- `ECHOAPI_RHBK_AUTH_MANUAL_STEPS.md` — Echo API RHBK Auth

## Integração SSO com RHBK (Keycloak em rhbk-gitops)

O 3scale usa o **Red Hat Build of Keycloak (RHBK)** já implantado no namespace `rhbk-gitops` (repositório [keycloak](https://github.com/luizfao/keycloak)) como provedor de identidade único para **Admin Portal**, **Developer Portal** e **APIs**. A configuração é feita no Admin Portal do 3scale e no console do Keycloak; não há parâmetros de SSO no APIManager CR.

**Pré-requisito:** Keycloak (RHBK) rodando em `rhbk-gitops` com um realm (ex.: `rhbk`) e a URL do realm acessível (ex.: `https://rhbk-rhbk-gitops.<cluster-domain>/realms/rhbk`).

**Importante (LDAP):** para usuários federados via LDAP (ex.: `ldaptest`) autenticarem corretamente no 3scale por SSO, o usuário precisa estar com **Email verified = true** no Keycloak. Em GitOps, o caminho recomendado é configurar `trustEmail=true` no provider LDAP do realm para evitar ajuste manual por usuário.

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
  - **Audience** → **Developer Portal** → **Settings** → **SSO Integrations**.
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

## Echo API OIDC (OAuth 2.0 Token Introspection)

Além do produto `Echo API` (autenticação por `user_key`), este repositório inclui um segundo produto **`Echo API OIDC`** que protege o mesmo backend com **OpenID Connect** e a policy de **OAuth 2.0 Token Introspection** do APIcast.

### Arquitetura

| Componente | Detalhes |
|-----------|---------|
| **Produto 3scale** | `echoapi-product-oidc` — autenticação OIDC, policy `token_introspection` |
| **Backend** | Mesmo `echoapi-backend` (`http://echoapi.echoapi-gitops.svc.cluster.local:9292`) |
| **IdP** | Keycloak `rhbk` realm (`rhbk-gitops`) |
| **Client Keycloak (Zync)** | `3scale-zync` — registra OIDC clients + introspecção de tokens |
| **Fluxo OAuth** | Client Credentials (`serviceAccountsEnabled: true`) |
| **Cache de tokens** | `max_cached_tokens: 1000`, `max_ttl_tokens: 60s` |

### O que é GitOps e o que é manual

**Via GitOps (Argo CD Application `3scale-echoapi`):**
- `Product` com OIDC config + policy `token_introspection`
- `Application` 3scale (`echoapi-application-oidc`)

**Manual (ver `ECHOAPI_OIDC_MANUAL_STEPS.md`):**
1. Criar o client `3scale-zync` no Keycloak (via Admin Portal ou `kcadm`)
2. Atribuir roles `manage-clients` + `view-clients` ao service account do `3scale-zync`
3. Atualizar o secret no `issuerEndpoint` de `echoapi-product-oidc.yaml` e sincronizar o Argo CD
4. Executar `ProxyConfigPromote` para publicar staging → produção
5. Obter `client_id`/`client_secret` da aplicação no Admin Portal do 3scale
6. Gerar Bearer Token via Client Credentials Flow e testar

### Pré-requisito: atualizar o issuerEndpoint com o secret real

O campo `issuerEndpoint` do produto contém as credenciais do cliente `3scale-zync` em formato URL:

```
https://3scale-zync:ZYNC_SECRET@rhbk-rhbk-gitops.<wildcardDomain>/realms/rhbk
```

Crie o client `3scale-zync` manualmente no Keycloak (passo 1 do `ECHOAPI_OIDC_MANUAL_STEPS.md`),
copie o secret gerado e substitua `change-me-zync-secret` no arquivo
`gitops/echoapi/echoapi-product-oidc.yaml` antes de sincronizar o `3scale-echoapi`.

### Comportamento da policy Token Introspection

- APIcast recebe a requisição com `Authorization: Bearer <JWT>`
- Chama o endpoint de introspecção do Keycloak: `/realms/rhbk/protocol/openid-connect/token/introspect`
- Keycloak retorna `{ "active": true/false }` — se `false`, APIcast rejeita com `Authentication Failed`
- Tokens são cacheados por até `max_ttl_tokens` segundos para reduzir chamadas ao Keycloak

---

## Echo API RHBK Auth (Anonymous + Token Introspection)

Terceiro produto, que implementa autenticação com qualquer usuário do Keycloak, sem necessidade de registro no 3scale.

### Arquitetura

| Componente | Detalhes |
|-----------|---------|
| **Produto 3scale** | `echoapi-product-rhbk-auth` — autenticação user_key + policies `default_credentials` + `token_introspection` |
| **Backend** | Mesmo `echoapi-backend` (`http://echoapi.echoapi-gitops.svc.cluster.local:9292`) |
| **IdP** | Keycloak `rhbk` realm (`rhbk-gitops`) |
| **Client Keycloak** | `4d045da9` — reutilizado para chamar o endpoint de introspeção (nenhum client novo necessário) |
| **Fluxo de autenticação** | Qualquer fluxo Keycloak (client credentials, password, etc.) |
| **Cache de tokens** | `max_cached_tokens: 100`, `max_ttl_tokens: 60s` |

### Como funciona

1. **`default_credentials`** — injeta `user_key: rhbk-anon-key` em todas as requests sem credenciais 3scale. Usada apenas para rate-limiting e tracking interno. (`anonymous_access` não existe como policy builtin no APIcast 3scale 2.16; `default_credentials` tem comportamento equivalente.)
2. **`token_introspection`** — valida o `Authorization: Bearer <JWT>` chamando `/realms/rhbk/protocol/openid-connect/token/introspect`. Se `{ "active": false }`, rejeita com `403`.
3. **`apicast`** — processamento padrão com a user_key injetada.

O cliente final nunca precisa saber a `user_key` do 3scale; envia apenas o seu Bearer token do Keycloak.

### O que é GitOps e o que é manual

**Via GitOps (Argo CD Application `3scale-echoapi`):**
- `Product` com policy chain `default_credentials` + `token_introspection` + `apicast`
- `Application` anônima (`echoapi-application-rhbk-anon`) + `ApplicationAuth` com `user_key: rhbk-anon-key`
- Routes `api-rhbk.example.com` e `api-rhbk-staging.example.com` (em `gitops/apicast-selfmanaged/apicast.yaml`)

**Manual (ver `ECHOAPI_RHBK_AUTH_MANUAL_STEPS.md`):**
1. Executar `ProxyConfigPromote` para publicar staging → produção
2. Reiniciar APIcast pods para carregar a nova configuração
3. Testar com Bearer token do Keycloak via `curl --resolve`

## Estrutura do repositório (manifestos)

| Caminho | Descrição |
|--------|------------|
| `bootstrap/0-repo-credentials-secret.example.yaml` | Exemplo de Secret para repositório privado (Argo CD) |
| `bootstrap/1-application-operator.yaml` | Argo CD Application `3scale-operator` (monitora `gitops/operator/`) |
| `bootstrap/2-application-databases.yaml` | Argo CD Application `3scale-databases` (monitora `gitops/databases/`) |
| `bootstrap/3-application-apimanager.yaml` | Argo CD Application `3scale-apimanager` — cluster primário (`gitops/apimanager/`) |
| `bootstrap/3-application-ha-apimanager.yaml` | Argo CD Application `3scale-ha-apimanager` — cluster HA (`gitops/apimanager-ha/`) |
| `bootstrap/4-application-apicast-selfmanaged.yaml` | Argo CD Application `3scale-apicast-selfmanaged` (`gitops/apicast-selfmanaged/`) |
| `bootstrap/5-application-echoapi.yaml` | Argo CD Application `3scale-echoapi` (`gitops/echoapi/`) |
| `gitops/operator/namespace.yaml` | Namespace `3scale-gitops` |
| `gitops/operator/operatorgroup.yaml` | OperatorGroup (operador no namespace) |
| `gitops/operator/subscription.yaml` | Subscription 3scale Operator 2.16 (aprovação Manual) |
| `gitops/databases/namespace.yaml` | Namespace `3scale-databases` |
| `gitops/databases/postgresql-system/postgresql.yaml` | PostgreSQL para System (`system_production`) em `3scale-databases` |
| `gitops/databases/redis-common/redis-config.yaml` | ConfigMap `redis-config-external` com persistência habilitada (`appendonly`, `save`) |
| `gitops/databases/redis-system/redis.yaml` | Redis para System (Sidekiq) em `3scale-databases` |
| `gitops/databases/redis-backend/redis.yaml` | Redis para Backend (storage + queues) em `3scale-databases` |
| `gitops/databases/3scale-secrets.yaml` | Secrets `system-database`, `system-redis`, `backend-redis` para o operador |
| `gitops/apimanager/apimanager.yaml` | APIManager CR — cluster primário (`wildcardDomain` + `externalComponents`) |
| `gitops/apimanager/debug-log-hook.yaml` | PostSync Job para habilitar logs de debug no backend-listener e system-app |
| `gitops/apimanager-ha/apimanager-ha.yaml` | APIManager CR — cluster HA (mesmo conteúdo, `wildcardDomain` diferente) |
| `gitops/apicast-selfmanaged/apicast.yaml` | `APIcast` CRs (production + staging) + Routes para todos os produtos |
| `gitops/apicast-selfmanaged/apicast-secret-init.yaml` | PreSync Job para popular o Secret `apicast-ha-portal-credentials` |
| `gitops/echoapi/namespace.yaml` | Namespace `echoapi-gitops` |
| `gitops/echoapi/echo-server.yaml` | Deploy do backend `echoapi` interno (ClusterIP) no namespace `echoapi-gitops` |
| `gitops/echoapi/3scale-capabilities.yaml` | CRs de capabilities: `Backend`, `Product` (user_key), `DeveloperAccount`, `DeveloperUser`, `Application` |
| `gitops/echoapi/echoapi-product-oidc.yaml` | `Product` OIDC + Token Introspection e `Application` correspondente |
| `gitops/echoapi/echoapi-product-rhbk-auth.yaml` | `Product` RHBK Auth (`default_credentials` + `token_introspection`) e `Application` anônima |
| `ECHOAPI_MANUAL_STEPS.md` | Passos manuais para `ProxyConfigPromote` e teste externo do Echo API (user_key) |
| `ECHOAPI_OIDC_MANUAL_STEPS.md` | Passos manuais para Echo API OIDC: secret Zync, roles Keycloak, token e teste |
| `ECHOAPI_RHBK_AUTH_MANUAL_STEPS.md` | Passos manuais para Echo API RHBK Auth: ProxyConfigPromote e teste com Bearer token Keycloak |

## Observações

- **2.16 e bancos externos**: a partir do 2.16, system database, system Redis e backend Redis são obrigatórios como componentes externos. Este repositório instala PostgreSQL (system) e Redis (system + backend) no namespace **`3scale-databases`** via GitOps; os secrets em `3scale-gitops` referenciam os serviços por FQDN (`*.3scale-databases.svc.cluster.local`). O APIManager usa `externalComponents` para referenciar esses secrets.
- **Zync database**: mantido em modo interno (gerenciado automaticamente pelo operador), conforme [suportado no 3scale 2.16](https://access.redhat.com/articles/2798521).
- **Senhas**: os secrets em `gitops/databases/` usam placeholders (`change-me`). Mantenha o `system-database` consistente com o secret `postgresql-system`.
- **Persistent volumes**: PostgreSQL usa StatefulSet+PVC RWO; Redis externo segue o padrão do guia de migração do 3scale (Deployment + PVC + ConfigMap `redis-config-external`) com `storageClassName: ocs-external-storagecluster-cephfs`.
- **Canal do operador**: `threescale-2.16`. Confirme no OperatorHub do cluster o nome do pacote e do canal se houver diferença.
- **Repositório Keycloak**: para implantar ou ajustar o RHBK usado no SSO (Admin, Developer Portal e APIs), use o repositório [keycloak](https://github.com/luizfao/keycloak) e o namespace `rhbk-gitops`.

---

## Interagindo com a API do 3Scale

### Token de acesso

Antes de interagir com a API do 3Scale é necessário obter a URL e o `access token`:

```bash
ACCESS_TOKEN=$(oc -n 3scale-gitops get secret system-seed \
  -o jsonpath='{.data.ADMIN_ACCESS_TOKEN}' | base64 -d)

ADMIN_URL="https://3scale-admin.$(oc get ingresscontroller -n openshift-ingress-operator \
  -o jsonpath='{.items[0].status.domain}')"
```

### Listar todos os produtos (services)

Utilize a chamada abaixo para listar todos os produtos, e obter o `SERVICE_ID`:

```bash
curl -s "${ADMIN_URL}/admin/api/services.json?access_token=${ACCESS_TOKEN}" \
  | python3 -c "import sys,json; [print(f\"ID: {s['service']['id']} | name: {s['service']['name']} | state: {s['service']['state']}\") for s in json.load(sys.stdin)['services']]"
```

*Importante:* Declare a variável `SEVICE_ID` com o ID obtido pelo comando acima (caso ocorra um erro, remova a linha do `python` para obter a resposta completa do 3scale).
```bash
SERVICE_ID=2
```

### Detalhes de um produto específico (por ID)

```bash
curl -s "${ADMIN_URL}/admin/api/services/${SERVICE_ID}.json?access_token=${ACCESS_TOKEN}" \
  | python3 -m json.tool
```

### Versão publicada de um produto por ambiente

```bash
# Staging (sandbox)

curl -s "${ADMIN_URL}/admin/api/services/${SERVICE_ID}/proxy/configs/sandbox/latest.json?access_token=${ACCESS_TOKEN}" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f\"version: {d['proxy_config']['version']}, env: {d['proxy_config']['environment']}\") "

# Production
curl -s "${ADMIN_URL}/admin/api/services/${SERVICE_ID}/proxy/configs/production/latest.json?access_token=${ACCESS_TOKEN}" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f\"version: {d['proxy_config']['version']}, env: {d['proxy_config']['environment']}\") "

```

### Obter todas as chaves das aplicações:

```bash
curl -s "${ADMIN_URL}/admin/api/applications.json?access_token=${ACCESS_TOKEN}" \
  | python3 -c "
import sys, json
apps = json.load(sys.stdin)['applications']
for a in apps:
    app = a['application']
    print(f\"id:            {app['id']}\")
    print(f\"name:          {app['name']}\")
    print(f\"user_key:      {app.get('user_key', 'N/A')}\")
    print(f\"client_id:     {app.get('client_id', 'N/A')}\")
    print(f\"client_secret: {app.get('client_secret', 'N/A')}\")
    print(f\"---\") "
```

---

## Melhorias

- Adicionar passos de "fork" para utilização e `contributing.md` para melhorias;

---

## Problemas Encontrados

### P1 — APIcast Operator não instalando (OperatorGroup duplicado)

**Problema:** Após adicionar a `Subscription` do operador APIcast junto à do operador 3scale (ambas no namespace `3scale-gitops`), o `InstallPlan` do APIcast nunca era aprovado e o operador não era instalado. O Argo CD não mostrava erro explícito no Application.

**Identificação:**

```bash
oc -n 3scale-gitops get operatorgroup
# Retornou: apicast-operator e 3scale-operator — dois OperatorGroups no mesmo namespace
oc -n 3scale-gitops describe operatorgroup apicast-operator
# Mensagem: "installs.operators.coreos.com "apicast-operator" is forbidden: only one OperatorGroup can exist per namespace"
```

O Kubernetes/OCP só permite **um** `OperatorGroup` por namespace. A presença de dois impedia o OLM de processar os `InstallPlan`s.

**Solução:** Removido o `OperatorGroup` `apicast-operator` de `gitops/operator/operatorgroup.yaml`, mantendo apenas o `3scale-operator`. O `OperatorGroup` existente cobre ambos os operadores. O `OperatorGroup` duplicado foi removido manualmente do cluster e o `InstallPlan` do APIcast foi aprovado.

---

### P2 — Colisão de hostnames entre produtos no APIcast Self-Managed

**Problema:** Após configurar os dois produtos (`echoapi-product` user_key e `echoapi-product-oidc`) para usar o mesmo APIcast Self-Managed (`api.example.com` / `api-staging.example.com`), ambos os produtos retornavam `403 Authentication failed`.

**Identificação:**

Os logs do APIcast revelaram:
```
skipping host api.example.com already defined by service 6
```

O APIcast só consegue rotear um hostname para **um** serviço. Como os dois produtos apontavam para os mesmos hostnames, o segundo serviço era ignorado. Todas as requisições eram roteadas para o primeiro serviço (id=2), cujas credenciais não coincidiam com as da requisição.

**Solução:** Atribuídos hostnames distintos a cada produto:

| Produto | Staging | Production |
|---------|---------|-----------|
| `echoapi-product` (user_key) | `api-staging.example.com` | `api.example.com` |
| `echoapi-product-oidc` (OIDC) | `api-oidc-staging.example.com` | `api-oidc.example.com` |

O `APIcast` CR cria Routes automaticamente somente para o seu `exposedHost`. Para o produto OIDC, foram criados Routes adicionais em `gitops/apicast-selfmanaged/apicast.yaml` apontando para os mesmos serviços (`apicast-apicast-ha-staging` e `apicast-apicast-ha-production`) com os novos hostnames. Os produtos foram re-promovidos via `ProxyConfigPromote`.

---

### P3 — Erro `403 Forbidden` Authentication failed para todos os serviços

**Problema:** Mesmo após corrigir a colisão de hostnames, o backend retornava `403` com a mensagem `service token "..." is invalid` para todos os service tokens e todos os service IDs. O problema estava sendo causado toda vez que o cluster reiniciava, mais especificamente as pods do `backend-redis`.

**Identificação:**

Verificação do Redis do backend (`backend-redis-external` no namespace `3scale-databases`):

```bash
oc -n 3scale-databases exec deployment/backend-redis-external -- \
  redis-cli keys "service_token*"
# Retornou: vazio — nenhuma chave service_token registrada
```

O Redis do backend só tinha chaves do tipo `service/id:X/state`, `service/id:X/provider_key` etc., mas nenhuma chave `service_token/*`. Os service tokens existiam corretamente no PostgreSQL (confirmado via Rails console: `Service.find(6).active_service_token`), mas nunca haviam sido sincronizados do banco relacional para o Redis do backend.

**Solução:** Executamos os passos da solution (3443351)[https://access.redhat.com/solutions/3443351] para forçar o sync.

```bash
oc -n 3scale-gitops exec -it deployment/system-app -- bash -c 'bundle exec rake backend:storage:rewrite'

oc -n 3scale-gitops rollout restart deployment/system-app
```

Isso fez com que as transações voltassem a funcionar, porém não corrigia a causa raiz do problema, que limpava o redis após o restart, a solução foi corrigir as configurações no `ConfigMap` `redis-config.yaml`:

| De | Para |
|---------|---------|
| save "" | save 900 1 |
|         | save 300 10 |
|         | save 60 10000 |
| appendonly no | appendonly yes |

---

### Como habilitar logs de debug

Os logs de debug são configurados via GitOps e aplicados ao cluster automaticamente:

**APIcast** — configurado diretamente no CR (`gitops/apicast-selfmanaged/apicast.yaml`):
```yaml
spec:
  logLevel: debug       # logs OpenResty/Lua
  oidcLogLevel: debug   # logs OIDC/JWT (somente staging)
```

**backend-listener e backend-worker** — o APIManager operator não expõe campo de log level; o PostSync hook `gitops/apimanager/debug-log-hook.yaml` aplica `CONFIG_LOG_LEVEL=debug` e reinicia os pods após cada sync do Application `3scale-apimanager`.

**system-app / sidekiq** — controlado pela chave `RAILS_LOG_LEVEL` no ConfigMap `system-environment` (gerenciado pelo operador). O mesmo PostSync hook atualiza este valor e reinicia os pods afetados.

Para desabilitar: altere os valores para `info` nos arquivos GitOps e force um novo sync.
