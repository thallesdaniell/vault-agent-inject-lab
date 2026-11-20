# Vault Agent Injector on Kubernetes

Laboratório de referência para entregar segredos do HashiCorp Vault a workloads Kubernetes pelo Vault Agent Injector. A aplicação recebe arquivos renderizados em `/vault/secrets`, autentica com uma ServiceAccount dedicada e acessa somente o caminho autorizado pela policy.

> Exemplo demonstrativo. Use Vault com TLS e storage apropriado fora de um laboratório. Nunca coloque valores reais de segredo em manifests, ConfigMaps, exemplos ou no Git.

## Objetivo

Este projeto mostra a integração entre Kubernetes auth, Vault policies e o webhook de injeção. Vault Agent autentica o pod e renderiza um segredo KV v2 em volume compartilhado em memória; a aplicação lê o arquivo sem receber o token do Vault.

```text
Pod com ServiceAccount dedicada
  └─ Vault Agent (init + sidecar) ── Kubernetes auth ──> Vault
             └─ renderiza /vault/secrets/database.json
                                      ▲
                         policy permite apenas demo/data/app
```

## Pré-requisitos

- Kubernetes com `kubectl` e Helm.
- Vault acessível pelo cluster com TLS verificado.
- Vault Agent Injector instalado, por exemplo pelo chart oficial `hashicorp/vault`.
- KV v2 habilitado no mount `secret`.

As versões do Vault, do chart e do Kubernetes devem ser fixadas e compatíveis com a plataforma que você usa. O laboratório não fixa uma instalação completa do Vault, porque isso depende do ambiente e da estratégia de storage.

## 1. Configurar Vault Kubernetes auth

Quando Vault roda dentro do Kubernetes, a documentação atual recomenda omitir `token_reviewer_jwt` e `kubernetes_ca_cert`: Vault lê e renova o token e o certificado locais. A ServiceAccount do Vault precisa de `system:auth-delegator` para chamar TokenReview. Execute a configuração de dentro de um pod Vault autorizado:

```sh
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}"
```

Se Vault estiver fora do cluster, configure explicitamente endereço do API server, CA e uma identidade reviewer com rotação planejada. Não copie tokens montados de um pod para um Secret de longa duração.

## 2. Criar policy e role

O caminho da policy KV v2 inclui `data/`. A role restringe autenticação a um namespace e ServiceAccount específicos:

```sh
vault policy write app-read - <<'HCL'
path "secret/data/demo/app" {
  capabilities = ["read"]
}
HCL

vault write auth/kubernetes/role/demo-app \
  bound_service_account_names="demo-app" \
  bound_service_account_namespaces="demo" \
  policies="app-read" \
  token_ttl="15m" \
  token_max_ttl="1h"
```

Crie no Vault o valor de demonstração separadamente, sem o registrar no Git:

```sh
vault kv put secret/demo/app username='demo-user' password='replace-me'
```

## 3. Instalar a workload

```sh
kubectl create namespace demo
kubectl apply -n demo -f examples/app.yaml
```

O exemplo usa `vault.hashicorp.com/agent-inject: "true"`, role dedicada e o caminho `secret/data/demo/app`. A annotation de template transforma a resposta KV em JSON. O arquivo montado tem permissão restrita e é entregue pelo volume compartilhado em memória do injector.

## Verificar a injeção

```sh
kubectl -n demo get pods
kubectl -n demo logs deploy/demo-app -c vault-agent
kubectl -n demo exec deploy/demo-app -- cat /vault/secrets/database.json
```

O último comando é apenas para verificar o laboratório. Em produção, não imprima segredos em logs ou terminais compartilhados; valide a aplicação lendo o arquivo sem exibir seu conteúdo.

## Decisões de segurança

- Use ServiceAccount própria para cada workload; nunca associe roles à `default`.
- Restrinja a policy a paths e operações exatos. Não conceda acesso a `secret/*`.
- Use tokens de curta duração e limite também `max_ttl` da role.
- Verifique TLS do Vault; não use `tls-skip-verify`.
- Não armazene credenciais ou arquivos renderizados em ConfigMaps.
- Proteja logs, dumps, backups e acesso `exec` aos pods, pois o segredo existe no filesystem do container.
- Considere políticas de rede, auditoria do Vault, rotação do segredo e comportamento da aplicação na renovação.

## Limitações

- O exemplo cobre KV v2 estático; não demonstra credenciais dinâmicas de banco nem rotação sem reinício da aplicação.
- O endereço do Vault e a instalação do injector variam conforme o cluster.
- O volume compartilhado em memória evita criar um Kubernetes Secret, mas processos autorizados no pod ainda podem ler os arquivos.

## Referências

- [Vault Agent Injector](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/injector)
- [Kubernetes auth method](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
- [Vault Injector annotations](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/injector/annotations)
- [Vault Helm chart](https://developer.hashicorp.com/vault/docs/platform/k8s/helm)
