# Laboratório Local (100% Offline / Custo Zero)

Este diretório contém tudo o que é necessário para rodar o projeto **Coworking Analytics** de forma totalmente local no seu computador, **sem precisar de conta na AWS, sem EKS e sem custos**.

---

## Opção 1: Via Docker Compose (Mais Rápido e Simples)

O Docker Compose é a maneira mais direta para quem trabalha com desenvolvimento e análise de dados. Ele sobe o banco PostgreSQL, executa os scripts SQL de semente (*seed*) automaticamente e inicia a API conectada.

### Como Executar:

1. Entre na pasta `local_lab`:
   ```bash
   cd local_lab
   ```

2. Inicie os serviços com um único comando:
   ```bash
   docker-compose up --build
   ```
   *(Ele construirá a imagem e iniciará o banco na porta `5433` e a API na porta `5153`).*

3. Em outro terminal, teste os endpoints da API:
   ```bash
   curl http://127.0.0.1:5153/api/reports/daily_usage
   curl http://127.0.0.1:5153/api/reports/user_visits
   ```

4. Para parar os serviços:
   ```bash
   docker-compose down
   ```

---

## Opção 2: Via Kubernetes Local (Minikube)

Para praticar exatamente os mesmos comandos e manifestos que usamos na AWS EKS, mas rodando no seu próprio computador.

### Passo 1: Instalar o Minikube (se ainda não tiver)
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

### Passo 2: Iniciar o Cluster Local
```bash
minikube start --driver=docker
```
*(Leva apenas 1 minuto! O seu `kubectl` passará a apontar automaticamente para o Minikube).*

### Passo 3: Subir o Banco PostgreSQL com Helm
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install postgresql bitnami/postgresql \
  --set auth.postgresPassword="localpassword123" \
  --set primary.persistence.enabled=false
```

### Passo 4: Executar os Scripts SQL no Banco
```bash
kubectl exec -i postgresql-0 -- env PGPASSWORD="localpassword123" psql -U postgres -d postgres < ../db/1_create_tables.sql
kubectl exec -i postgresql-0 -- env PGPASSWORD="localpassword123" psql -U postgres -d postgres < ../db/2_seed_users.sql
kubectl exec -i postgresql-0 -- env PGPASSWORD="localpassword123" psql -U postgres -d postgres < ../db/3_seed_tokens.sql
```

### Passo 5: Construir e Carregar a Imagem no Minikube
```bash
# 1. Constrói a imagem localmente
docker build -t coworking-analytics:local ../analytics

# 2. Envia a imagem diretamente para dentro do Minikube (dispensa o AWS ECR!)
minikube image load coworking-analytics:local
```

### Passo 6: Aplicar os Manifestos do Kubernetes
```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Verifique se os pods estão rodando:
```bash
kubectl get pods
```

### Passo 7: Testar os Endpoints
```bash
# Abre o túnel
kubectl port-forward svc/local-coworking-analytics 5153:5153 &

# Testa as rotas
curl http://127.0.0.1:5153/api/reports/daily_usage
curl http://127.0.0.1:5153/api/reports/user_visits
```

### Para pausar ou desligar o Minikube quando terminar:
```bash
minikube stop
# Ou para apagar o cluster do seu disco:
# minikube delete
```
