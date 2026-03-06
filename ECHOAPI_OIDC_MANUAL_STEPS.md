# Echo API OIDC — passos manuais

Este arquivo descreve os passos necessários para colocar o produto `Echo API OIDC`
em funcionamento. O produto em si é gerenciado via GitOps (`gitops/echoapi/echoapi-product-oidc.yaml`),
mas as etapas de infraestrutura no Keycloak são feitas manualmente para evitar
conflitos com clients já existentes no realm.

## Pré-requisitos

- Cluster acessível via `oc login`
- Acesso ao Admin Portal do Keycloak em `https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/admin/rhbk/console/`
- Argo CD Application `3scale-echoapi` ainda **não** sincronizado com este produto (faça o sync após o passo 2)

---

## 1) Criar o cliente 3scale-zync no Keycloak (manual)

O cliente `3scale-zync` é usado pelo Zync (componente do 3scale) para registrar
automaticamente clientes OIDC no Keycloak, e pelo APIcast para chamar o endpoint
de introspecção de tokens.

### Via Admin Portal do Keycloak

1. Acesse: `https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/admin/rhbk/console/`
2. Vá em **Clients** → **Create client**
3. Preencha:
   - **Client type**: `OpenID Connect`
   - **Client ID**: `3scale-zync`
   - **Name**: `3scale Zync OIDC Integration`
4. Clique em **Next**
5. Na tela **Capability config**:
   - Habilite **Client authentication**
   - Desabilite **Standard flow**
   - Desabilite **Direct access grants**
   - Habilite **Service accounts roles**
6. Clique em **Next** → **Save**
7. Na aba **Credentials**:
   - Copie o **Client secret** gerado (ou clique em **Regenerate** para obter um novo)
   - Anote este valor — será usado no próximo passo

### (Ainda não testado) Via kcadm (alternativa CLI)

```bash
KEYCLOAK_URL="https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io"
REALM="rhbk"
ADMIN_PASS="$(oc -n rhbk-gitops get secret rhbk-initial-admin -o jsonpath='{.data.password}' | base64 -d)"

oc exec -n rhbk-gitops deployment/rhbk -- \
  /opt/keycloak/bin/kcadm.sh config credentials \
    --server "${KEYCLOAK_URL}" --realm master \
    --user admin --password "${ADMIN_PASS}"

oc exec -n rhbk-gitops deployment/rhbk -- \
  /opt/keycloak/bin/kcadm.sh create clients -r "${REALM}" \
    -s clientId=3scale-zync \
    -s name="3scale Zync OIDC Integration" \
    -s enabled=true \
    -s clientAuthenticatorType=client-secret \
    -s standardFlowEnabled=false \
    -s implicitFlowEnabled=false \
    -s directAccessGrantsEnabled=false \
    -s serviceAccountsEnabled=true \
    -s publicClient=false \
    -s protocol=openid-connect

# Obter o secret gerado automaticamente
oc exec -n rhbk-gitops deployment/rhbk -- \
  /opt/keycloak/bin/kcadm.sh get clients -r "${REALM}" \
    --fields clientId,secret \
    --query "clientId=3scale-zync" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['secret'])"
```

---

## 2) Atribuir roles ao service account do 3scale-zync

Após criar o client, atribua as roles `manage-clients` e `view-clients` do
`realm-management` ao service account, para que o Zync possa criar/atualizar
clients OIDC no Keycloak automaticamente.

### Via Admin Portal do Keycloak

1. Em **Clients** → `3scale-zync` → aba **Service accounts roles**
2. Clique em **Assign role**
3. No filtro, selecione **Filter by clients** e escolha `realm-management`
4. Selecione as roles:
   - `manage-clients`
   - `view-clients`
5. Clique em **Assign**

### (Ainda não testado) Via kcadm (alternativa CLI)

```bash
KEYCLOAK_URL="https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io"
REALM="rhbk"
ADMIN_USER="admin"
ADMIN_PASS="$(oc -n rhbk-gitops get secret rhbk-initial-admin -o jsonpath='{.data.password}' | base64 -d)"

# Login
oc exec -n rhbk-gitops deployment/rhbk -- \
  /opt/keycloak/bin/kcadm.sh config credentials \
    --server "${KEYCLOAK_URL}" \
    --realm master \
    --user "${ADMIN_USER}" \
    --password "${ADMIN_PASS}"

# Obter o ID do service account do client 3scale-zync
SA_USER_ID=$(oc exec -n rhbk-gitops deployment/rhbk -- \
  /opt/keycloak/bin/kcadm.sh get clients -r "${REALM}" \
    --fields clientId,serviceAccountUserId \
    --query "clientId=3scale-zync" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['serviceAccountUserId'])")

echo "Service Account User ID: ${SA_USER_ID}"

# Obter o ID do client realm-management
RM_CLIENT_ID=$(oc exec -n rhbk-gitops deployment/rhbk -- \
  /opt/keycloak/bin/kcadm.sh get clients -r "${REALM}" \
    --fields id,clientId \
    --query "clientId=realm-management" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])")

echo "realm-management client ID: ${RM_CLIENT_ID}"

# Obter IDs das roles manage-clients e view-clients
for ROLE in manage-clients view-clients; do
  ROLE_ID=$(oc exec -n rhbk-gitops deployment/rhbk -- \
    /opt/keycloak/bin/kcadm.sh get clients/${RM_CLIENT_ID}/roles/${ROLE} -r "${REALM}" \
    | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")
  echo "Assigning ${ROLE} (${ROLE_ID}) to service account ${SA_USER_ID}"
  oc exec -n rhbk-gitops deployment/rhbk -- \
    /opt/keycloak/bin/kcadm.sh create users/${SA_USER_ID}/role-mappings/clients/${RM_CLIENT_ID} \
      -r "${REALM}" \
      -b "[{\"id\":\"${ROLE_ID}\",\"name\":\"${ROLE}\"}]"
  echo "Role ${ROLE} atribuída."
done
```

---

## 3) Atualizar o secret no manifesto GitOps e sincronizar

Com o secret do client `3scale-zync` obtido no passo 1, atualize o campo
`issuerEndpoint` em `gitops/echoapi/echoapi-product-oidc.yaml`:

```yaml
issuerEndpoint: "https://3scale-zync:<SEU_SECRET_AQUI>@rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/realms/rhbk"
```

Faça commit, push e sincronize o Argo CD Application `3scale-echoapi`.

---

## 4) Aguardar sincronização dos CRs no 3scale

```bash
oc -n 3scale-gitops wait --for=condition=Synced --timeout=300s \
  product/echoapi-product-oidc

oc -n 3scale-gitops wait --for=condition=Ready --timeout=300s \
  application.capabilities.3scale.net/echoapi-application-oidc
```

Verificar status detalhado:

```bash
oc -n 3scale-gitops describe product echoapi-product-oidc
oc -n 3scale-gitops describe application.capabilities.3scale.net echoapi-application-oidc
```

---

## 5) Promover o proxy (ProxyConfigPromote)

O 3scale não publica automaticamente as configurações de staging para produção.
Aplique o seguinte CR uma única vez após a sincronização acima:

```bash
cat <<EOF | oc apply -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: ProxyConfigPromote
metadata:
  name: echoapi-oidc-promote
  namespace: 3scale-gitops
spec:
  productCRName: echoapi-product-oidc
  production: true
  deleteCR: true
EOF
```

> `deleteCR: true` faz o CR se auto-deletar após a promoção (evita re-tentativas).

---

## 6) Obter credenciais OIDC da aplicação (client_id e client_secret)

O `ApplicationAuth` não é compatível com produtos OIDC. As credenciais devem ser
obtidas diretamente no Admin Portal do 3scale após o Zync sincronizar a aplicação
com o Keycloak.

**Via Admin Portal do 3scale:**
1. Acesse: `https://3scale-admin.apps.cluster-zrdcz.dynamic.redhatworkshops.io`
2. Vá em **Audience** → **Accounts** → `Echo API Org` → **Applications**
3. Clique em `Echo API OIDC App`
4. Anote o **Client ID** e o **Client Secret** mostrados em **API Credentials**

**Via Admin Portal do Keycloak (confirmar):**
1. Acesse: `https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/admin/rhbk/console/`
2. Vá em **Clients** — procure pelo client_id obtido no passo anterior

Opcional: armazenar as credenciais no cluster para uso em scripts:

```bash
oc -n 3scale-gitops create secret generic echoapi-product-oidc-credentials \
  --from-literal=ApplicationID='<CLIENT_ID>' \
  --from-literal=ApplicationKey='<CLIENT_SECRET>' \
  --dry-run=client -o yaml | oc apply -f -
```

---

## 7) Obter um Bearer Token para teste (Client Credentials Flow)

Com o `client_id` e `client_secret` obtidos no passo 6, gere um token via Client Credentials:

```bash
KEYCLOAK_URL="https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io"
REALM="rhbk"
CLIENT_ID="<client_id da aplicação>"
CLIENT_SECRET="<client_secret da aplicação>"

# Ou, se guardou na Secret opcional do passo 6:
# CLIENT_ID=$(oc -n 3scale-gitops get secret echoapi-product-oidc-credentials \
#   -o jsonpath='{.data.ApplicationID}' | base64 -d)
# CLIENT_SECRET=$(oc -n 3scale-gitops get secret echoapi-product-oidc-credentials \
#   -o jsonpath='{.data.ApplicationKey}' | base64 -d)

TOKEN=$(curl -s -X POST \
  "${KEYCLOAK_URL}/realms/${REALM}/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=${CLIENT_ID}" \
  -d "client_secret=${CLIENT_SECRET}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "TOKEN=${TOKEN}"
```

---

## 8) Testar o Echo API OIDC via APIcast

```bash
# Staging
curl -v -H "Authorization: Bearer ${TOKEN}" \
  "https://echoapi-oidc-3scale-apicast-production.apps.cluster-zrdcz.dynamic.redhatworkshops.io/"

# Produção (após ProxyConfigPromote)
curl -v -H "Authorization: Bearer ${TOKEN}" \
  "https://echoapi-oidc-3scale-apicast-production.apps.cluster-zrdcz.dynamic.redhatworkshops.io/"
  
```

Resposta esperada do echoapi (200 OK com JSON do request):

```json
{
  "method": "GET",
  "path": "/",
  "args": "",
  "body": "",
  "headers": { ... }
}
```

**Erros comuns:**

| Erro | Causa provável | Solução |
|------|---------------|---------|
| `Authentication Failed` | Token inativo ou inválido | Verifique se o token não expirou e se as credenciais estão corretas |
| `No Mapping Rule matched` | Rota não mapeada | Verifique os `mappingRules` no produto |
| `503 / upstream connect error` | Zync ainda não criou o client OIDC | Aguarde ou verifique os logs do Zync |
| `Token not active` na introspecção | Token expirado ou emitido para realm/client diferente | Gere um novo token com as credenciais corretas |
