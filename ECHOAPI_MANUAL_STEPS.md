# Echo API - passos manuais (o que nao e 100% GitOps)

Este arquivo cobre os passos operacionais que nao sao ideais para reconciliacao continua via Argo CD.

## 1) Aplicar o Application do Echo API

```bash
oc apply -f bootstrap/5-application-echoapi.yaml
```

No Argo CD, sincronize o app `3scale-echoapi`.

O upstream `echoapi` roda no namespace `echoapi-gitops`.

## 2) Aguardar sincronizacao dos CRs de capabilities

```bash
oc -n 3scale-gitops wait --for=condition=Synced --timeout=300s backend/echoapi-backend
oc -n 3scale-gitops wait --for=condition=Synced --timeout=300s product/echoapi-product
oc -n 3scale-gitops wait --for=condition=Ready --timeout=300s developeraccount/echoapi-account
oc -n 3scale-gitops wait --for=condition=Ready --timeout=300s application.capabilities.3scale.net/echoapi-application-v2
```

## 3) Promover proxy config (staging e production)

`ProxyConfigPromote` e um CR de acao "one-shot" (nao e bom manter em reconciliacao continua).
Execute manualmente sempre que houver alteracoes relevantes no `Product`:

```bash
oc -n 3scale-gitops apply -f - <<'EOF'
apiVersion: capabilities.3scale.net/v1beta1
kind: ProxyConfigPromote
metadata:
  name: echoapi-promote-staging
spec:
  productCRName: echoapi-product
  production: false
  deleteCR: true
EOF
```

```bash
oc -n 3scale-gitops apply -f - <<'EOF'
apiVersion: capabilities.3scale.net/v1beta1
kind: ProxyConfigPromote
metadata:
  name: echoapi-promote-production
spec:
  productCRName: echoapi-product
  production: true
  deleteCR: true
EOF
```

## 4) Definir e verificar credenciais via ApplicationAuth (GitOps)

O `ApplicationAuth` `echoapi-product-appauth` define o `UserKey` da aplicação `echoapi-application-v2`
a partir da Secret `echoapi-product-appauth`. Antes de sincronizar:

1. Edite `gitops/echoapi/3scale-capabilities.yaml` → Secret `echoapi-product-appauth`
2. Substitua `change-me-user-key` pelo valor desejado (mínimo 5 caracteres)
3. Faça commit, push e sincronize o Argo CD Application `3scale-echoapi`

Após o sync, verifique se o `ApplicationAuth` foi aplicado com sucesso:

```bash
oc -n 3scale-gitops get applicationauth echoapi-product-appauth -o yaml
```

O status esperado após sucesso:

```yaml
status:
  conditions:
    - status: "True"
      type: Ready
      message: "Application authentication has been successfully pushed..."
```

Recuperar o `UserKey` efetivo a qualquer momento:

```bash
oc -n 3scale-gitops get secret echoapi-product-appauth \
  -o jsonpath='{.data.UserKey}' | base64 -d; echo
```

## 5) Testar chamada externa

O produto `Echo API` usa autenticação por `user_key`. Passe a chave via query string `?user_key=`.

### Obter o IP do router OpenShift

```bash
ROUTER_IP=$(nslookup \
  "$(oc get route console -n openshift-console -o jsonpath='{.spec.host}')" \
  | awk '/^Address/ && !/#/ {print $2}')
echo "Router IP: ${ROUTER_IP}"
```

### Ler a user_key do Secret GitOps

```bash
USER_KEY=$(oc -n 3scale-gitops get secret echoapi-product-appauth \
  -o jsonpath='{.data.UserKey}' | base64 -d)
echo "User Key: ${USER_KEY}"
```

### Testar nos 2 endpoints

```bash
# Staging
curl -s --resolve "api-staging.example.com:80:${ROUTER_IP}" \
  "http://api-staging.example.com/?user_key=${USER_KEY}" \
  -w "\nHTTP: %{http_code}\n"

# Production
curl -s --resolve "api.example.com:80:${ROUTER_IP}" \
  "http://api.example.com/?user_key=${USER_KEY}" \
  -w "\nHTTP: %{http_code}\n"
```

Resposta esperada: **HTTP 200** com o corpo JSON do Echo API.
