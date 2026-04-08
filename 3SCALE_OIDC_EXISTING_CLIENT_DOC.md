# 3scale OIDC com Cliente Keycloak Existente

Esta documentação descreve a solução implementada para criar e gerenciar Produtos e Aplicações no Red Hat 3scale (versão 2.16) utilizando autenticação OIDC (OpenID Connect) associada a um cliente (`client_id` e `client_secret`) **já existente** no Keycloak.

A integração do 3scale com o Keycloak utilizando clientes pré-existentes exige que o 3scale conheça exatamente o `client_id` e o `client_secret` que a aplicação consumidora utilizará para gerar seus tokens. Como o CRD `Application` do Operador 3scale não suporta a injeção manual destas credenciais OIDC, a solução divide a responsabilidade:

1. **Criação das Entidades Base (Operador):** O Operador do 3scale continua sendo responsável por criar e reconciliar o **3scale Product**, os **Application Plans**, as **Mapping Rules** e as **Policies**. O Produto é configurado com a URL do Issuer (Keycloak) para que o APIcast saiba onde validar os tokens.
2. **Criação da Aplicação (Account Management API):** Em vez de usar um CRD `Application`, utilizamos um **Job Kubernetes** (executado como um hook do ArgoCD na fase após o `sync`). Este Job executa um script Python que consome a Account Management API do 3scale. O script realiza as buscas necessárias para encontrar o Produto e o Plano corretos e, em seguida, faz uma chamada `POST` na API do 3Scale criando a aplicação e forçando a injeção do `application_id` (`client_id` do Keycloak) e `application_key` (`client_secret` do Keycloak), garantindo a integridade dos dados mesmo se o `Application` já existir. Como o Job possui a anotação `BeforeHookCreation`, ele permanece no cluster após a execução, permitindo que você inspecione seu estado e seus logs entre um sync e outro do ArgoCD.

Desta forma, garantimos que a aplicação no 3scale esteja perfeitamente sincronizada com o cliente existente no Keycloak, permitindo que os tokens gerados pelo Keycloak sejam validados com sucesso pelo APIcast.

*Nota:* a `client_secret` não deve ser alterada, somente se extremamente necessário. Ela só pode ser alterada pela interface gráfica do 3Scale (outras formas não surtirão efeito), caso isso seja feito, é necessário obtê-la pela interface e atualizar na Secret (do passo 4.2) para que o Job execute a validação com sucesso.

---

## 1. Configurando o Secret do Issuer OIDC (`myapi-oidc-issuer`)

O 3scale precisa saber como se comunicar com o Keycloak (o "Issuer") para validar tokens e gerenciar o realm. A URL do issuer deve conter as credenciais de um cliente confidencial do Keycloak com permissões para gerenciar o realm (frequentemente o `3scale-zync`, mas pode ser outro).

Você deve popular o Secret `myapi-oidc-issuer` com a chave `issuerEndpoint`. O formato esperado da URL é:
`https://CLIENT_ID:CLIENT_SECRET@KEYCLOAK_HOST/realms/REALM`

**Exemplo de Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapi-oidc-issuer
  namespace: 3scale-gitops
type: Opaque
stringData:
  issuerEndpoint: "https://3scale-zync:SUA_SECRET_AQUI@rhbk-rhbk-gitops.apps.cluster.com/realms/rhbk"
```
*Nota: Se o `client_id` ou `client_secret` possuírem caracteres especiais, eles devem ser codificados usando percent-encoding (RFC 3986).*

---

## 2. Configurando o Secret de Gerenciamento do 3scale (`myapi-oidc-3scale-management`)

O Job de registro da aplicação precisa se autenticar na Account Management API do 3scale. Para isso, ele utiliza o Secret `myapi-oidc-3scale-management`.

Este é o mesmo formato de Secret de credenciais de Tenant (`tenant-credentials`) utilizado pelo operador. Ele contém a URL de administração do 3scale e um Access Token com permissões de leitura e escrita na Account Management API.

**Você pode utilizar um Secret existente.** Só é necessário criar um novo caso você esteja utilizando o Tenant padrão (`admin`) criado durante a instalação do 3scale, ou se preferir isolar as credenciais do Job.

**Exemplo de Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapi-oidc-3scale-management
  namespace: 3scale-gitops
type: Opaque
stringData:
  adminURL: "https://3scale-admin.apps.cluster.com"
  token: "SEU_ACCESS_TOKEN_AQUI"
```

---

## 3. O ConfigMap do Script de Registro (`myapi-oidc-register-application-script`)

Para manter o código limpo e reutilizável, o script Python que faz a chamada à API do 3scale foi extraído para o ConfigMap `myapi-oidc-register-application-script`.

**De forma resumida, este script:**
1. Lê as variáveis de ambiente fornecidas pelo Job (credenciais da API, dados do produto, plano, conta e credenciais do Keycloak).
2. Busca o `account_id` da Developer Account no 3scale.
3. Verifica se a aplicação já existe (buscando por `application_id` ou nome). Se existir, ele apenas valida se o `client_secret` no 3scale coincide com o esperado e finaliza com sucesso (idempotência).
4. Busca o `service_id` do Produto e o `plan_id` do Application Plan.
5. Executa um `POST` na API do 3scale (`/admin/api/accounts/{account_id}/applications.json`) criando a aplicação e forçando o `application_id` (Client ID) e `application_key[]` (Client Secret) desejados.

Este ConfigMap é genérico e não precisa ser alterado ao criar novas aplicações; ele apenas armazena a lógica que o Job executará.

---

## 4. Passo a Passo para Criar Novos Produtos e Aplicações

Para criar um novo conjunto de Produto e Aplicação apontando para um cliente Keycloak existente, siga os passos abaixo:

### 4.1. Criando um novo 3scale Product
Crie o manifesto do `Product` referenciando o Secret do Issuer.

**Importante:** Se o cliente do 3scale que gerencia o realm (o Issuer) for diferente para este novo produto, você precisará criar e referenciar um **novo Secret** (ex: `novo-api-oidc-issuer`) em vez de reutilizar o `myapi-oidc-issuer`.

```yaml
apiVersion: capabilities.3scale.net/v1beta1
kind: Product
metadata:
  name: novo-product-oidc-existing
spec:
  name: Novo API OIDC
  systemName: novo_oidc_existing
  deployment:
    apicastSelfManaged:
      stagingPublicBaseURL: "http://novo-oidc-staging.example.com"
      productionPublicBaseURL: "http://novo-oidc.example.com"
      authentication:
        oidc:
          issuerType: keycloak
          issuerEndpointRef:
            name: myapi-oidc-issuer # Atualize se usar outro client de gerência no Keycloak
          authenticationFlow:
            serviceAccountsEnabled: true
          jwtClaimWithClientID: azp
          jwtClaimWithClientIDType: plain
  # ... (backendUsages, mappingRules, policies)
  applicationPlans:
    basic-novo-oidc:
      name: Basic Novo OIDC
```

**Atenção aos Nomes:**
O `name` do Product em si pode ser qualquer um, mas o `systemName` do Product e o `systemName` do Application Plan (neste caso `basic-novo-oidc`) serão referenciados no próximo passo. Além disso, **o nome da Application** que você deseja criar no 3scale será a base para nomear a Secret e o Job na etapa a seguir.

### 4.2. Criando uma nova 3scale Application (via Job)

Como a aplicação será criada via API, você precisará de dois recursos com o nome da nova aplicação: um **Secret** com os dados e o **Job** que executará o script.

**A) Crie o Secret da Aplicação:**
Este Secret armazenará o Client ID e Client Secret do Keycloak, além dos metadados necessários para o script localizar os recursos no 3scale.

**Importante:** O valor da propriedade `name` (em `metadata.name`) deste Secret **deve ser exatamente o mesmo nome** definido na propriedade `appName` (o nome da Application que será criada no 3scale).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: novo-product-oidc-existing-application # Mesmo valor de appName
  namespace: 3scale-gitops
type: Opaque
stringData:
  clientId: "CLIENT_ID_DO_KEYCLOAK"
  clientSecret: "CLIENT_SECRET_DO_KEYCLOAK"
  accountOrgName: "Echo API Org"
  productSystemName: "novo_oidc_existing" # systemName do Product criado no passo 4.1
  planSystemName: "basic-novo-oidc"       # systemName do Application Plan do passo 4.1
  appName: "novo-product-oidc-existing-application" # O nome da Application no 3scale
  appDescription: "Criada via API com client Keycloak existente"
```

**B) Crie o Job de Registro:**
Crie o Job para executar o script. Este Job é configurado como um hook do ArgoCD para executar na fase `PostSync` (após a criação do Product e Plan). A anotação `BeforeHookCreation` garante que o Job permaneça no cluster após a execução, permitindo que você verifique o estado e os logs da criação da Application entre as sincronizações. Ele só será apagado automaticamente quando o ArgoCD iniciar um novo sync.

**Importante:** 
1. O valor da propriedade `name` (em `metadata.name`) do Job **deve ser exatamente o mesmo nome** definido na propriedade `appName` (o nome da Application).
2. O valor de `name` dentro de cada `secretKeyRef` deve fazer referência **à Secret criada no passo anterior (passo A)**.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: novo-product-oidc-existing-application # Mesmo valor de appName
[...]
spec:
  template:
    spec:
      containers:
[...]
            # Referências para a Secret da aplicação criada no passo A:
            - name: ACCOUNT_ORG_NAME
              valueFrom: { secretKeyRef: { name: novo-product-oidc-existing-application, key: accountOrgName } }
            - name: PRODUCT_SYSTEM_NAME
              valueFrom: { secretKeyRef: { name: novo-product-oidc-existing-application, key: productSystemName } }
            - name: PLAN_SYSTEM_NAME
              valueFrom: { secretKeyRef: { name: novo-product-oidc-existing-application, key: planSystemName } }
            - name: APP_NAME
              valueFrom: { secretKeyRef: { name: novo-product-oidc-existing-application, key: appName } }
            - name: APP_DESCRIPTION
              valueFrom: { secretKeyRef: { name: novo-product-oidc-existing-application, key: appDescription } }
            - name: CLIENT_ID
              valueFrom: { secretKeyRef: { name: novo-product-oidc-existing-application, key: clientId } }
            - name: CLIENT_SECRET
              valueFrom: { secretKeyRef: { name: novo-product-oidc-existing-application, key: clientSecret } }
[...]
```