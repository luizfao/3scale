# Echo API OIDC — passos manuais

Este arquivo descreve os passos necessários para colocar o produto `Echo API OIDC`
em funcionamento. A maior parte é gerenciada via GitOps (`gitops/echoapi/echoapi-product-oidc.yaml`),
mas algumas etapas exigem interação direta com o Keycloak e o 3scale Admin Portal.

## Pré-requisitos

- Argo CD Application `3scale-echoapi` sincronizado (inclui `echoapi-product-oidc.yaml`)
- Keycloak (`rhbk`) sincronizado com o cliente `3scale-zync` criado
- Cluster acessível via `oc login`

---

## 1) Definir o secret do cliente 3scale-zync no Keycloak

O cliente `3scale-zync` é criado via GitOps no realm `rhbk` com o secret
`change-me-zync-secret`. Antes de sincronizar, defina um valor seguro:

1. Abra o arquivo `keycloak/gitops/rhbk/realm-import/realm.yaml`
2. Substitua `change-me-zync-secret` por um valor gerado:
   ```bash
   openssl rand -hex 20
   ```
3. Atualize também `gitops/echoapi/echoapi-product-oidc.yaml` → campo `issuerEndpoint`,
   substituindo `change-me-zync-secret` pelo mesmo valor.
4. Faça commit, push e sincronize os dois Argo CD Applications:
   - **keycloak** (para criar o client no Keycloak)
   - **3scale-echoapi** (para criar o produto no 3scale)

---

## 2) Atribuir roles ao service account do 3scale-zync no Keycloak

Após o Keycloak sincronizar e o cliente `3scale-zync` ser criado, é necessário
atribuir as roles de gerenciamento de clientes ao service account dele. Isso permite
que o Zync registre/atualize automaticamente os clientes OIDC criados pelos Applications
do 3scale.

### Via Admin Portal do Keycloak

1. Acesse: `https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/admin/rhbk/console/`
2. Vá em **Clients** → `3scale-zync` → aba **Service accounts roles**
3. Clique em **Assign role**
4. No filtro, selecione **Filter by clients** e escolha `realm-management`
5. Selecione as roles:
   - `manage-clients`
   - `view-clients`
6. Clique em **Assign**

### Via kcadm (opcional, se preferir CLI)

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
done
```

---

## 3) Aguardar sincronização dos CRs no 3scale

```bash
oc -n 3scale-gitops wait --for=condition=Synced --timeout=300s \
  product/echoapi-product-oidc

oc -n 3scale-gitops wait --for=condition=Ready --timeout=300s \
  application.capabilities.3scale.net/echoapi-application-oidc
```

Verificar status detalhado:

```bash
oc -n 3scale-gitops describe product echoapi-product-oidc
oc -n 3scale-gitops describe application echoapi-application-oidc
```

---

## 4) Promover o proxy (ProxyConfigPromote)

O 3scale não publica automaticamente as configurações de staging para produção.
Aplique o seguinte CR uma única vez após a sincronização acima:

```bash
cat <<EOF | oc apply -f -
apiVersion: capabilities.3scale.net/v1alpha1
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

## 5) Obter credenciais OIDC da aplicação (client_id e client_secret)

Após o Zync sincronizar, o 3scale cria automaticamente um OIDC client no Keycloak
para a aplicação `echoapi-application-oidc`. Para obter as credenciais:

**Via Admin Portal do 3scale:**
1. Acesse: `https://3scale-admin.apps.cluster-zrdcz.dynamic.redhatworkshops.io`
2. Vá em **Audience** → **Accounts** → `Echo API Org` → **Applications**
3. Clique em `Echo API OIDC App`
4. Anote o **Client ID** e o **Client Secret** mostrados em **API Credentials**

**Via Admin Portal do Keycloak (confirmar):**
1. Acesse: `https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io/admin/rhbk/console/`
2. Vá em **Clients** — procure pelo client_id obtido no passo anterior

---

## 6) Obter um Bearer Token para teste (Client Credentials Flow)

Com o `client_id` e `client_secret` da aplicação, obtenha um token via client credentials:

```bash
CLIENT_ID="<client_id da aplicação echoapi-application-oidc>"
CLIENT_SECRET="<client_secret da aplicação>"
KEYCLOAK_URL="https://rhbk-rhbk-gitops.apps.cluster-zrdcz.dynamic.redhatworkshops.io"
REALM="rhbk"

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

## 7) Testar o Echo API OIDC via APIcast

```bash
# Staging
curl -v -H "Authorization: Bearer ${TOKEN}" \
  "https://echoapi-3scale-apicast-staging.apps.cluster-zrdcz.dynamic.redhatworkshops.io/"

# Produção (após ProxyConfigPromote)
curl -v -H "Authorization: Bearer ${TOKEN}" \
  "https://echoapi-3scale-apicast-production.apps.cluster-zrdcz.dynamic.redhatworkshops.io/"
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
