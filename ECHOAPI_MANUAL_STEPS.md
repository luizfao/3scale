# Echo API - passos manuais (o que nao e 100% GitOps)

Este arquivo cobre os passos operacionais que nao sao ideais para reconciliacao continua via Argo CD.

## 1) Aplicar o Application do Echo API

```bash
oc apply -f bootstrap/application-echoapi.yaml
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

## 5) Descobrir endpoint publico e testar chamada externa

O endpoint publico do produto e exibido no 3scale Admin Portal:

- Products -> Echo API -> Integration -> Configuration
- copie a URL de staging ou production

Com a URL e uma chave de app (por exemplo `user_key`), teste:

```bash
curl -i "https://<public-echoapi-url>/?user_key=<USER_KEY>"
```

Se seu produto estiver em `app_id/app_key`, ajuste os parametros:

```bash
curl -i "https://<public-echoapi-url>/?app_id=<APP_ID>&app_key=<APP_KEY>"
```
