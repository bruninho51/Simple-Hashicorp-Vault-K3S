# HashiCorp Vault no K3s

Este diretório contém a configuração para rodar o **HashiCorp Vault** em um cluster K3s instalado em uma VPS.

O Vault é configurado para rodar com **armazenamento persistente**, exposto via **Ingress** com TLS automático (cert-manager + Traefik), utilizando **Deployments**, **Persistent Volumes** e **Ingress** para expor a interface.
Além disso, inclui a configuração do **Vault Agent Injector**, responsável por injetar secrets em pods automaticamente via mutating webhook.

---

## 📂 Estrutura de arquivos

### Vault

```
k8s/vault/
├── 01-namespace.yaml
├── 02-serviceAccount.yaml
├── 03-clusterRoleBinding.yaml
├── 04-pvc.yaml
├── 05-deployment.yaml
├── 06-service.yaml
└── 07-ingress.yaml
```

### Vault Agent Injector

```
k8s/agent-injector/
├── 01-issuerCertificateCA.yaml
├── 02-certificateCA.yaml
├── 03-issuerCertificateWebhook.yaml
├── 04-certificateWebhook.yaml
├── 05-serviceAccount.yaml
├── 06-clusterRole.yaml
├── 07-clusterRoleBinding.yaml
├── 08-deployment.yaml
├── 09-service.yaml
└── 10-mutatingWebhookConfiguration.yaml
```

---

## 📄 Descrição dos manifests

### Vault

#### 01-namespace.yaml

Cria o **namespace `vault`**.
Todos os recursos do Vault serão isolados dentro deste namespace.

#### 02-serviceAccount.yaml

Cria um **ServiceAccount** para o Vault dentro do namespace `vault`.
Usado futuramente para autenticação com Kubernetes e injeção de secrets.

#### 03-clusterRoleBinding.yaml

Cria um **ClusterRoleBinding** que concede permissão ao ServiceAccount do Vault para realizar **TokenReview** no cluster.
Isso é necessário para o Vault validar JWTs de outros pods.

#### 04-pvc.yaml

Cria um **PersistentVolumeClaim (PVC)** chamado `vault-data` para armazenamento persistente:

* Usa `local-path` como `storageClass`.
* Reserva **10GiB** de espaço.
* Montado em `/vault/data` no container.

#### 05-deployment.yaml

Cria o **Deployment** do Vault:

* Imagem oficial: `hashicorp/vault:1.18.1`.
* Criação dinâmica do arquivo `/tmp/vault.hcl` com:

  * `ui = true` → habilita interface web.
  * `disable_mlock = true` → evita erro de bloqueio de memória.
  * `storage "file"` → usa o PVC `/vault/data`.
  * `listener "tcp"` → porta `8200`, sem TLS (TLS tratado pelo Ingress).
  * `api_addr = "https://vault.orcamentos.app"` → endereço público.
* Probes de **liveness** e **readiness** verificam `/v1/sys/health`.
* Usa o ServiceAccount definido no passo 02 para interações com Kubernetes.

#### 06-service.yaml

Cria um **Service** interno chamado `vault`:

* Porta: `8200` (HTTP).
* Tipo: `ClusterIP` (acesso interno).

#### 07-ingress.yaml

Cria um **Ingress** chamado `vault-ingress` para acesso externo:

* Controlador: **Traefik** (`kubernetes.io/ingress.class: traefik`).
* TLS automático via **cert-manager** (`letsencrypt-prod`).
* Redireciona tráfego HTTPS para o Service `vault` na porta `8200`.

---

### Vault Agent Injector

#### 01-issuerCertificateCA.yaml

Cria um **Issuer self-signed** no namespace `vault` para gerar a CA raiz que será usada para assinar certificados internos.

#### 02-certificateCA.yaml

Cria um **Certificate** autoassinado usando o Issuer anterior, servindo como CA raiz para os certificados do webhook.

#### 03-issuerCertificateWebhook.yaml

Cria um **Issuer baseado na CA criada** no passo anterior, usado para assinar certificados do webhook.

#### 04-certificateWebhook.yaml

Cria o **Certificate do webhook** que o Vault Agent Injector vai usar:

* O Secret gerado será montado no container.
* Duration: 1 ano (`8760h`) com renovação automática 30 dias antes (`720h`).
* Certificado autoassinado será rotacionado automaticamente pelo **cert-manager**.

#### 05-serviceAccount.yaml

Cria o **ServiceAccount** do Vault Agent Injector dentro do namespace `vault`.

#### 06-clusterRole.yaml

Define as permissões necessárias para o Vault Agent Injector:

* Acessar pods, logs, serviceaccounts.
* Atualizar `mutatingwebhookconfigurations`.

#### 07-clusterRoleBinding.yaml

Associa o **ClusterRole** ao ServiceAccount do injector.

#### 08-deployment.yaml

Cria o **Deployment do Vault Agent Injector**:

* Monta o Secret TLS do webhook.
* Configura args/env para apontar para o Vault (`VAULT_ADDR`) e TLS.
* Probes de **readiness** e **liveness**.
* Limites e requests de CPU/memória.

#### 09-service.yaml

Cria um **Service** interno para expor o webhook.

#### 10-mutatingWebhookConfiguration.yaml

Cria a **MutatingWebhookConfiguration** que permite injeção automática de secrets em pods:

* Exclui namespaces `vault` e `kube-system`.
* Annotation `cert-manager.io/inject-ca-from` para atualizar o CA bundle automaticamente.
* Operações: `CREATE` e `UPDATE` para pods.

---

## 🚀 Como aplicar

1. Aplicar **manifests do Vault**:

```bash
kubectl apply -f k8s/vault/
```

2. Aplicar **manifests do Vault Agent Injector**:

```bash
kubectl apply -f k8s/agent-injector/
```

> ⚠️ A ordem dos manifests na pasta `agent-injector` garante que a CA seja criada antes do certificado do webhook, e que todos os recursos necessários estejam disponíveis antes do deployment do injector.

---

## 🔍 Verificação

### Vault

1. **Verificar pods:**

```bash
kubectl get pods -n vault
```

Exemplo:

```
NAME                     READY   STATUS    RESTARTS   AGE
vault-xxxxxx-yyyyy       1/1     Running   0          1m
```

2. **Verificar logs:**

```bash
kubectl logs -n vault deployment/vault
```

3. **Checar status:**

```bash
kubectl exec -n vault -it deploy/vault -- vault status
```

4. **Acessar interface web:**

```
https://vault.orcamentos.app
```

### Vault Agent Injector

1. **Verificar pods:**

```bash
kubectl get pods -n vault -l app=vault-agent-injector
```

2. **Logs do injector:**

```bash
kubectl logs -n vault deploy/vault-agent-injector
```

3. **Verificar MutatingWebhookConfiguration:**

```bash
kubectl get mutatingwebhookconfiguration vault-agent-injector-cfg -o yaml
```

---

## 📌 Próximos passos

1. Inicializar o Vault:

```bash
kubectl exec -n vault -it deploy/vault -- vault operator init
```

2. Deslacrar (unseal) com as chaves geradas:

```bash
kubectl exec -n vault -it deploy/vault -- vault operator unseal
```

3. Autenticar no Vault:

```bash
kubectl exec -n vault -it deploy/vault -- vault login <ROOT_TOKEN>
```

4. Habilitar Kubernetes Auth:

```bash
kubectl exec -n vault -it deploy/vault -- vault auth enable kubernetes
```

5. Configurar Kubernetes Auth usando ServiceAccount:

```bash
kubectl exec -n vault -it deploy/vault -- vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
```

## 🧪 Testando a injeção de secrets em pod (`.env`) com múltiplas chaves

### 1️⃣ Criar secret no Vault

```bash
kubectl exec -n vault -it deploy/vault -- vault kv put secret/dummy \
  segredo1="valor_super_secreto_1" \
  segredo2="valor_super_secreto_2"
```

* `secret/data/dummy` é o caminho do KV v2.
* `segredo1` e `segredo2` são os campos que serão injetados.

---

### 2️⃣ Criar policy no Vault

```bash
kubectl exec -n vault -it deploy/vault -- vault policy write dummy-policy - <<EOF
path "secret/data/dummy" {
  capabilities = ["read"]
}
EOF
```

* Permite **ler** o segredo `secret/data/dummy`.

---

### 3️⃣ Criar role Kubernetes no Vault

```bash
kubectl exec -n vault -it deploy/vault -- vault write auth/kubernetes/role/app-role \
  bound_service_account_names="default" \
  bound_service_account_namespaces="teste" \
  policies="dummy-policy" \
  ttl="1h"
```

---

### 4️⃣ Criar namespace de teste

```bash
kubectl create namespace teste
```

---

### 5️⃣ Criar pod de teste com Vault Agent Injector e template `.env`

```bash
sudo kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vault-test-read
  namespace: teste
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "app-role"
    vault.hashicorp.com/agent-inject-secret-my_secret: "secret/data/dummy"
    vault.hashicorp.com/agent-inject-template-.env: |
      {{- with secret "secret/data/dummy" -}}
      SEGREDO_1={{ .Data.data.segredo1 }}
      SEGREDO_2={{ .Data.data.segredo2 }}
      {{- end }}
spec:
  serviceAccountName: default
  containers:
  - name: test
    image: busybox
    command: ["sh", "-c"]
    args: ["cat /vault/secrets/.env && sleep 3600"]
EOF
```

* O Vault Agent Injector cria automaticamente o arquivo `/vault/secrets/.env`.
* As variáveis `SEGREDO_1` e `SEGREDO_2` são preenchidas a partir do secret.

---

### 6️⃣ Verificar se funcionou

```bash
kubectl logs -n teste vault-test-read
```

Saída esperada:

```
SEGREDO_1=valor_super_secreto_1
SEGREDO_2=valor_super_secreto_2
```

