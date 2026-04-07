# Echo API RHBK Auth — Passos Manuais

Produto: **Echo API RHBK Auth** (`echoapi-product-rhbk-auth`)
Arquivo GitOps: `gitops/echoapi/echoapi-product-rhbk-auth.yaml`

---

## O que este produto faz

Este produto implementa o padrão **Default Credentials + OAuth 2.0 Token Introspection**:

- O cliente envia **apenas** `Authorization: Bearer <token_keycloak>` — nenhuma `user_key` 3scale é necessária na chamada.
- Qualquer usuário com conta válida no Keycloak (realm `rhbk`) pode usar a API, sem precisar ser cadastrado no 3scale.
- A policy `default_credentials` injeta uma `user_key` interna transparente (da aplicação `echoapi-application-rhbk-anon`), usada pelo 3scale apenas para rate-limiting e tracking.
- A policy `token_introspection` valida o Bearer JWT chamando o endpoint de introspeção do Keycloak. Se o token não for ativo (`"active": false`), a requisição é rejeitada com `403`.

### Diferença em relação aos outros produtos

| Produto | Autenticação | O que o cliente envia |
|---------|-------------|----------------------|
| `echoapi-product` | API Key (user_key) | `?user_key=<key>` |
| `echoapi-product-oidc` | OIDC (client credentials) | `Authorization: Bearer <JWT>` via client 3scale |
| `echoapi-product-rhbk-auth` | Bearer JWT de qualquer usuário Keycloak | `Authorization: Bearer <JWT>` do próprio usuário |

---

## Pré-requisitos

1. O Argo CD Application `3scale-echoapi` sincronizado com sucesso (produtos criados no 3scale).
2. Keycloak (RHBK) rodando em `rhbk-gitops` com realm `rhbk` e usuários configurados.
3. O client `4d045da9` já existente no realm `rhbk` (reutilizado para introspeção — nenhum client novo é necessário).
4. Routes `apicast-ha-production-rhbk` e `apicast-ha-staging-rhbk` criados (via sync do Application `3scale-apicast-selfmanaged`).

---

## Passos após o sync do Argo CD

### 1. Verificar que o produto e a aplicação foram criados

```bash
oc -n 3scale-gitops get product echoapi-product-rhbk-auth \
  -o jsonpath='{.status.conditions[?(@.type=="Synced")].status}' && echo

oc -n 3scale-gitops get application.capabilities.3scale.net echoapi-application-rhbk-anon \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' && echo
```

Ambos devem retornar `True`.

### 2. Verificar que a user_key anônima foi configurada

```bash
oc -n 3scale-gitops get applicationauth echoapi-rhbk-anon-appauth \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' && echo
```

Deve retornar `True`. Se retornar `False` com `ApplicationKey limit reached`, delete o ApplicationAuth e reaplique:

```bash
oc -n 3scale-gitops delete applicationauth echoapi-rhbk-anon-appauth
oc apply -f gitops/echoapi/echoapi-product-rhbk-auth.yaml
```

### 3. Promover a configuração de staging para produção

```bash
cat <<EOF | oc apply -f -
apiVersion: capabilities.3scale.net/v1beta1
kind: ProxyConfigPromote
metadata:
  name: echoapi-product-rhbk-auth-promote
  namespace: 3scale-gitops
spec:
  productCRName: echoapi-product-rhbk-auth
  production: true
EOF

# Aguardar conclusão
sleep 10
oc -n 3scale-gitops get proxyconfigpromote echoapi-product-rhbk-auth-promote \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' && echo
```

### 4. (Opcional) Reiniciar os pods do APIcast

O APIcast Self-Managed recarrega a configuração automaticamente via polling. O restart manual só é necessário se a nova configuração não aparecer após alguns minutos.

```bash
oc -n 3scale-gitops rollout restart \
  deployment/apicast-apicast-ha-staging \
  deployment/apicast-apicast-ha-production

oc -n 3scale-gitops rollout status deployment/apicast-apicast-ha-staging --timeout=60s
```

---

## Testando a API

### 1.Obter o IP do router OpenShift

```bash
ROUTER_IP=$(nslookup \
  "$(oc get route console -n openshift-console -o jsonpath='{.spec.host}')" \
  | awk '/^Address/ && !/#/ {print $2}')
echo "Router IP: ${ROUTER_IP}"
```

### 2.Obter um `CLIENT_ID` e `CLIENT_SECRET` do RHBK

**Via Admin Portal do Keycloak:**
1. Acesse: `https://rhbk-rhbk-gitops.apps.cluster-bcb52.dynamic.redhatworkshops.io/admin/master/console/#/rhbk`
2. Vá em **Clients** — procure por um client
3. Vá em **Credentials** — clique no botão para revelar a secret para confirmar

**Via API do 3scale utilizando de um produto OIDC criado anteriormente**
```bash
ACCESS_TOKEN=$(oc -n 3scale-gitops get secret system-seed \
  -o jsonpath='{.data.ADMIN_ACCESS_TOKEN}' | base64 -d)
ADMIN_URL="https://3scale-admin.$(oc get ingresscontroller \
  -n openshift-ingress-operator -o jsonpath='{.items[0].status.domain}')"

curl -s "${ADMIN_URL}/admin/api/applications.json?access_token=${ACCESS_TOKEN}" \
  | python3 -c "
import sys, json
apps = json.load(sys.stdin)['applications']
for a in apps:
    app = a['application']
    if 'OIDC' in app.get('name',''):
        print(f\"name:          {app['name']}\")
        print(f\"client_id:     {app.get('client_id', 'N/A')}\")
        print(f\"client_secret: {app.get('client_secret', 'N/A')}\")"
```

Salvar o `CLIENT_ID` e `CLIENT_SECRET` em uma variável para utilizar nos próximos comandos:
```bash
CLIENT_ID=client_id
CLIENT_SECRET=client_secret
``` 

### 3.Obter um Bearer token do Keycloak (qualquer usuário do realm)

**Opção A — Client Credentials (service account):**

```bash
KEYCLOAK_URL="https://rhbk-rhbk-gitops.apps.cluster-bcb52.dynamic.redhatworkshops.io"
REALM=rhbk
#CLIENT_ID= # Obtido no comando anterior
#CLIENT_SECRET= # Obtido no comando anterior

TOKEN=$(curl -s -X POST "${KEYCLOAK_URL}/realms/${REALM}/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${CLIENT_ID}&client_secret=${CLIENT_SECRET}" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

echo "Token obtido: $([ -n "$TOKEN" ] && echo 'OK' || echo 'FALHOU')"
```

**Opção B — Resource Owner Password (usuário LDAP, ex.: ldaptest):**

```bash
KEYCLOAK_URL="https://rhbk-rhbk-gitops.apps.cluster-bcb52.dynamic.redhatworkshops.io"
#CLIENT_ID= # Obtido no comando anterior
#CLIENT_SECRET= # Obtido no comando anterior

TOKEN=$(curl -s -X POST "${KEYCLOAK_URL}/realms/rhbk/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password&client_id=${CLIENT_ID}&client_secret=${CLIENT_SECRET}&username=ldaptest&password=<senha_ldap>" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

> **Nota:** Para usar o fluxo de Resource Owner Password, o client precisa ter `directAccessGrantsEnabled: true` no Keycloak.

### 4.Testar via staging

```bash
curl -s --resolve "api-rhbk-staging.example.com:80:${ROUTER_IP}" \
  -H "Authorization: Bearer ${TOKEN}" \
  "http://api-rhbk-staging.example.com/" \
  -w "\nHTTP: %{http_code}\n"
```

Esperado: **HTTP 200** com o corpo JSON do Echo API.

### 5.Testar via produção

```bash
curl -s --resolve "api-rhbk.example.com:80:${ROUTER_IP}" \
  -H "Authorization: Bearer ${TOKEN}" \
  "http://api-rhbk.example.com/" \
  -w "\nHTTP: %{http_code}\n"
```

### Testar rejeição de token inválido

```bash
curl -s --resolve "api-rhbk-staging.example.com:80:${ROUTER_IP}" \
  -H "Authorization: Bearer token-invalido" \
  "http://api-rhbk-staging.example.com/" \
  -w "\nHTTP: %{http_code}\n"
# Esperado: HTTP 403 Authentication failed
```

### Testar rejeição sem token

```bash
curl -s --resolve "api-rhbk-staging.example.com:80:${ROUTER_IP}" \
  "http://api-rhbk-staging.example.com/" \
  -w "\nHTTP: %{http_code}\n"
# Esperado: HTTP 403 (token_introspection tenta introspeccionar nil → rejeita)
```

---

## Troubleshooting

### `403 Authentication failed` com token válido

Verificar logs do APIcast em modo debug:

```bash
oc -n 3scale-gitops logs deployment/apicast-apicast-ha-staging --tail=50 \
  | grep -E "introspect|token|auth|403|active"
```

Causas comuns:
- **`token not active`**: token expirado ou revogado no Keycloak — obtenha um novo token.
- **`failed to execute token introspection. status: 4xx`**: problema com as credenciais do client de introspeção (`client_id`/`client_secret`) — verifique o client `4d045da9` no Keycloak.
- **`skipping host ... already defined`**: conflito de hostname com outro produto — verifique se `api-rhbk*.example.com` é único na configuração.

### ProxyConfigPromote não Ready

```bash
oc -n 3scale-gitops describe proxyconfigpromote echoapi-product-rhbk-auth-promote
```

Se a mensagem for `cannot promote to production as no product changes detected`, o produto não foi atualizado desde a última promoção. Force um re-sync do Argo CD ou faça uma alteração mínima no produto.

---
