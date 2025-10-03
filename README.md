# HashiCorp Vault no K3s

Este diretório contém a configuração para rodar o **HashiCorp Vault** em um cluster K3s instalado em uma VPS.
O Vault é configurado para rodar com **armazenamento persistente**, exposto via **Ingress** com TLS automático (cert-manager + Traefik), utilizando **Deployments**, **Persistent Volumes** e **Ingress** para expor a interface.

---

## 📂 Estrutura de arquivos

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

---

## 📄 Descrição dos manifests

### 01-namespace.yaml

Cria o **namespace `vault`**.
Todos os recursos do Vault serão isolados dentro deste namespace.

### 02-serviceAccount.yaml

Cria um **ServiceAccount** para o Vault dentro do namespace `vault`.
Isso é usado futuramente para autenticação com o Kubernetes e injeção de secrets.

### 03-clusterRoleBinding.yaml

Cria um **ClusterRoleBinding** que concede permissão ao ServiceAccount do Vault para realizar **TokenReview** no cluster.
Isso é necessário para o Vault validar JWTs de outros pods.

### 04-pvc.yaml

Cria um **PersistentVolumeClaim (PVC)** chamado `vault-data` para armazenamento persistente:

* Usa `local-path` como `storageClass`.
* Reserva **10GiB** de espaço.
* Montado em `/vault/data` no container.

### 05-deployment.yaml

Cria o **Deployment** do Vault com as seguintes configurações:

* Imagem oficial: `hashicorp/vault:1.18.1`.
* Configuração dinâmica do arquivo `/tmp/vault.hcl` com:

  * `ui = true` → habilita interface web.
  * `disable_mlock = true` → evita erro de bloqueio de memória no container.
  * `storage "file"` → usa o PVC `/vault/data` como backend.
  * `listener "tcp"` → expõe na porta `8200`, sem TLS (TLS tratado pelo Ingress).
  * `api_addr = "https://vault.orcamentos.app"` → endereço público do Vault.
* **Probes de saúde** (`livenessProbe` e `readinessProbe`) verificam `/v1/sys/health`.
* Usa o ServiceAccount definido no passo 02 para interações com Kubernetes.

### 06-service.yaml

Cria um **Service** chamado `vault` para expor o Vault internamente:

* Porta: `8200` (HTTP).
* Tipo: `ClusterIP` (acesso interno).

### 07-ingress.yaml

Cria um **Ingress** chamado `vault-ingress` para acesso externo:

* Controlador: **Traefik** (`kubernetes.io/ingress.class: traefik`).
* TLS automático via **cert-manager** (`letsencrypt-prod`).
* Redireciona tráfego HTTPS para o Service `vault` na porta `8200`.

---

## 🚀 Como aplicar

Aplicar todos os manifests de uma vez:

```bash
kubectl apply -f k8s/vault/
```

Isso criará namespace, ServiceAccount, ClusterRoleBinding, PVC, Deployment, Service e Ingress.

---

## 🔍 Verificação

1. **Verificar pods:**

```bash
kubectl get pods -n vault
```

Exemplo de saída:

```
NAME                     READY   STATUS    RESTARTS   AGE
vault-xxxxxx-yyyyy       1/1     Running   0          1m
```

2. **Verificar logs do Vault:**

```bash
kubectl logs -n vault deployment/vault
```

3. **Checar status via CLI:**

```bash
kubectl exec -n vault -it deploy/vault -- vault status
```

Exemplo de saída:

```
Seal Type          shamir
Initialized        false
Sealed             true
...
```

> Significa que o Vault está pronto para inicialização.

4. **Acessar interface web:**

```
https://vault.orcamentos.app
```

---

## 📌 Próximos passos

1. **Inicializar o Vault:**

```bash
kubectl exec -n vault -it deploy/vault -- vault operator init
```

2. **Deslacrar (unseal) com as chaves geradas:**

```bash
kubectl exec -n vault -it deploy/vault -- vault operator unseal
```

3. **Autenticar no Vault:**

```bash
kubectl exec -n vault -it deploy/vault -- vault login <ROOT_TOKEN>
```

4. **Habilitar Kubernetes Auth:**

```bash
kubectl exec -n vault -it deploy/vault -- vault auth enable kubernetes
```

5. **Configurar Kubernetes Auth usando ServiceAccount:**

```bash
kubectl exec -n vault -it deploy/vault -- vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc.cluster.local" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
```