# Red Hat 3scale API Management 2.16 via OpenShift GitOps (Argo CD)

Este repositório contém manifests **GitOps** para instalar Red Hat 3scale API Management **2.16** em um cluster OpenShift, em um namespace dedicado, com integração de SSO opcional ao **Keycloak (RHBK)** já implantado no namespace `rhbk-gitops` (repositório [keycloak](https://github.com/luizfao/keycloak)).

Documentação do produto: [Red Hat 3scale API Management 2.16](https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.16/)

## O que é implantado

- **Namespace**: `3scale-gitops`
- **3scale Operator** (OLM) no canal `threescale-2.16`, aprovação de Install Plan **Manual**
- **APIManager** CR: instância 3scale com domínio configurável (`wildcardDomain`)

A integração com **SSO (Keycloak)** é feita **depois** da instalação, no Admin Portal do 3scale (por produto: OpenID Connect / Red Hat Single Sign-On), apontando para o Keycloak no namespace `rhbk-gitops`.

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

### 4) Bootstrap do Argo CD

Aplique o `Application` que aponta para `gitops/`:

```bash
oc apply -f bootstrap/application.yaml
```

Isso cria o Application **3scale** no Argo CD, que sincroniza o conteúdo de `gitops/` no namespace `3scale-gitops`.

### 5) Aprovar o Install Plan do operador (Manual)

Após o sync, o Subscription do 3scale pode ficar aguardando aprovação. Liste e aprove o Install Plan:

```bash
oc -n 3scale-gitops get installplans
oc -n 3scale-gitops edit installplan <nome-do-installplan>
# Altere spec.approved para true
# Ou: oc -n 3scale-gitops patch installplan <nome> -p '{"spec":{"approved":true}}' --type=merge
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

## Integração SSO com Keycloak (RHBK em rhbk-gitops)

O 3scale não configura SSO via APIManager CR; a integração é feita no **Admin Portal** do 3scale e no **Keycloak** (RHBK) já rodando no namespace `rhbk-gitops` (repositório [keycloak](https://github.com/luizfao/keycloak)).

Resumo:

1. **Keycloak (rhbk-gitops)**  
   - URL do realm (ex.: `https://rhbk-rhbk-gitops.<cluster-domain>/realms/rhbk`).  
   - Criar um **client** para o 3scale (Zync) com permissões de “Realm Management” (ex.: `manage-clients`) para o 3scale sincronizar aplicações.

2. **3scale Admin Portal**  
   - Em cada **Product** (ou no nível da conta): **Integration** → **Configuration** → **Authentication**.  
   - Escolher **OpenID Connect**, tipo **Red Hat Single Sign-On**.  
   - Configurar:
     - **OpenID Connect Issuer**: URL do realm do Keycloak (ex.: `https://rhbk-rhbk-gitops.<cluster-domain>/realms/rhbk`).  
     - **Zync client** (client ID e secret criados no Keycloak).  
   - Após alterar o modo de autenticação, pode ser necessário **recriar aplicações** para o Zync sincronizar credenciais com o Keycloak.

Referência: [Securing APIs using OIDC with Red Hat Single Sign-On](https://docs.redhat.com/en/documentation/red_hat_3scale_api_management/2.16/html-single/admin_portal_guide/) (Admin Portal / Authentication).

## Estrutura do repositório (manifestos)

| Caminho | Descrição |
|--------|------------|
| `bootstrap/application.yaml` | Argo CD Application (monitora `gitops/`) |
| `bootstrap/repo-credentials-secret.example.yaml` | Exemplo de Secret para repositório privado |
| `gitops/namespace.yaml` | Namespace `3scale-gitops` |
| `gitops/operator/operatorgroup.yaml` | OperatorGroup (operador no namespace) |
| `gitops/operator/subscription.yaml` | Subscription 3scale Operator 2.16 (Manual) |
| `gitops/apimanager/apimanager.yaml` | APIManager CR (definir `wildcardDomain`) |

## Observações

- **Persistent volumes**: o operador 3scale provisiona os PVCs necessários (RWX para portal, RWO para Redis/MySQL) quando não se usam bancos externos.
- **Canal do operador**: `threescale-2.16`. Confirme no OperatorHub do cluster o nome do pacote e do canal se houver diferença (ex.: `3scale-operator`).
- **Repositório Keycloak**: para implantar ou ajustar o Keycloak usado no SSO, use o repositório [keycloak](https://github.com/luizfao/keycloak) e o namespace `rhbk-gitops`.
