
# HashiCorp Vault no K3S

Este diretório contém uma configuração para rodar o HashiCorp Vault em um cluster K3s instalado em uma VPS.
O Vault é configurado para rodar com armazenamento persistente, exposto via Ingress com TLS automático (cert-manager + Traefik), utilizando **Deployments**, **Persistent Volumes** e **Ingress** para expor a interface.

## Estrutura de Arquivos

```
k8s/vault/
├── 01-namespace.yaml
├── 02-pvc.yaml
├── 03-deployment.yaml
├── 04-service.yaml
└── 05-ingress.yaml
```

### 📄 01-namespace.yaml

Cria o **namespace `vault`**.
Todos os recursos do Vault serão isolados dentro deste namespace.

### 📄 02-pvc.yaml

Cria um **PersistentVolumeClaim (PVC)** chamado `vault-data`.
Esse PVC garante armazenamento persistente para os dados do Vault, evitando perda de informações caso o pod seja reiniciado.

* Usa `local-path` como `storageClass`.
* Reserva **10GiB** de espaço.
* Montado em `/vault/data` no container.

### 📄 03-deployment.yaml

Cria um **Deployment** chamado `vault`.
Esse é o recurso principal que roda o Vault no cluster.

Configurações importantes:

* Usa a imagem oficial `hashicorp/vault:1.18.1`.
* Cria o arquivo `/tmp/vault.hcl` dinamicamente e inicia o Vault em modo servidor (`vault server -config=/tmp/vault.hcl`).
* Configurações no `vault.hcl`:

  * `ui = true` → habilita a interface web.
  * `disable_mlock = true` → evita erro de bloqueio de memória no container.
  * `storage "file"` → usa o PVC (`/vault/data`) como backend de armazenamento.
  * `listener "tcp"` → expõe na porta `8200`, sem TLS (TLS tratado pelo Ingress).
  * `api_addr = "https://vault.orcamentos.app"` → endereço público do Vault.
* **Probes de saúde** (`livenessProbe` e `readinessProbe`) verificam se a API responde em `/v1/sys/health`.
* Usa o **ServiceAccount padrão** do Kubernetes (`default`) para poder interagir com o cluster futuramente (ex: secrets injection).

### 📄 04-service.yaml

Cria um **Service** chamado `vault`.
Serve como ponto de rede interno no cluster, permitindo que outros pods acessem o Vault via `vault.vault.svc.cluster.local:8200`.

* Porta exposta: `8200` (HTTP).
* Tipo: `ClusterIP` (interno ao cluster).

### 📄 05-ingress.yaml

Cria um **Ingress** chamado `vault-ingress`.
Expõe o Vault externamente para acesso via domínio `vault.orcamentos.app`.

* Usa o **Traefik** como controlador de ingress (`kubernetes.io/ingress.class: traefik`).
* Integra com **cert-manager** para TLS automático via `letsencrypt-prod`.
* Redireciona tráfego HTTPS para o Service `vault` na porta `8200`.

---

## 🚀 Como aplicar

Aplicar todos os manifests de uma vez:

```bash
kubectl apply -f k8s/vault/
```

Isso criará namespace, PVC, Deployment, Service e Ingress.

---

## 🔍 Como verificar se está funcionando

1. **Verificar pods:**

   ```bash
   kubectl get pods -n vault
   ```

   Você deve ver algo como:

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

   Esperado:

   ```
   Seal Type          shamir
   Initialized        false
   Sealed             true
   ...
   ```

   > Isso significa que o Vault está pronto para ser inicializado.

4. **Acessar interface web:**
   Abra no navegador:

   ```
   https://vault.orcamentos.app
   ```

---

## 📌 Próximos passos

* **Inicializar o Vault**:

  ```bash
  kubectl exec -n vault -it deploy/vault -- vault operator init
  ```

* **Deslacrar (unseal) com as keys geradas**:

  ```bash
  kubectl exec -n vault -it deploy/vault -- vault operator unseal
  ```

* **Autenticar no Vault**:

  ```bash
  kubectl exec -n vault -it deploy/vault -- vault login <ROOT_TOKEN>
  ```
