# Documentação Completa - CI/CD Azure DevOps + Kubernetes + S3

Guia detalhado para configurar CI/CD com templates reutilizáveis para backend (Go + Kubernetes) e frontend (Node.js + S3).

---

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Backend: Go + Kubernetes](#backend-go--kubernetes)
3. [Frontend: Node.js + S3 + CloudFront](#frontend-nodejs--s3--cloudfront)
4. [Estrutura do Repositório](#estrutura-do-repositório)
5. [Service Connections](#service-connections)
6. [Variáveis de Ambiente](#variáveis-de-ambiente)
7. [Pipeline de Configuração](#pipeline-de-configuração)
8. [Agent - Pré-Requisitos](#agent---pré-requisitos)

---

## 🏗️ Visão Geral

### Fluxo Backend (Go)
```
Branch: stage/main
    ↓
Trigger Pipeline
    ↓
SAST (Trivy + SonarQube)
    ↓
Build Docker (ECR Private)
    ↓
Push ECR
    ↓
Deploy Kubernetes (stage/prod)
```

### Fluxo Frontend (Node.js)
```
Branch: stage/main
    ↓
Trigger Pipeline
    ↓
SAST (Trivy + SonarQube)
    ↓
Build (npm run build)
    ↓
Push S3
    ↓
Invalidate CloudFront
```

---

## 🔙 Backend: Go + Kubernetes

### Estrutura do Projeto

```
seu-servico-api/
├── ci/
│   └── pipeline.yml
├── cd/
│   ├── prod/
│   │   └── deployment.yaml
│   └── stage/
│       └── deployment.yaml
├── cmd/
│   └── main.go
├── internal/
│   ├── handler/
│   ├── service/
│   ├── model/
│   └── repository/
├── pkg/
│   └── utils/
├── Dockerfile
├── Makefile
├── go.mod
├── go.sum
├── .gitignore
└── README.md
```

### Dockerfile

```dockerfile
# Múltiplos estágios para reduzir tamanho da imagem
FROM YOUR_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/golang:1.23.0-alpine3.20 AS builder

WORKDIR /app

COPY go.mod go.sum ./

RUN apk add --no-cache upx git make \
    && go env -w GOPRIVATE=github.com/seu-org/seu-repo \
    && go mod download

COPY . .

RUN rm -rf .env

# Build com flags de otimização
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-s -w" \
    -o main ./cmd/main.go

# Comprimir binary com upx
RUN upx main

# Final stage - apenas essencial
FROM YOUR_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/alpine:3.12.1

WORKDIR /root/

# Copiar binary do builder
COPY --from=builder /app/main .

# Dependências mínimas
RUN apk add --no-cache ca-certificates tzdata

EXPOSE 8080

CMD ["./main"]
```

**Detalhes importantes:**
- Imagem base vem do ECR privado (não Docker Hub)
- Build multi-stage reduz tamanho final
- `GOPRIVATE` para módulos privados
- `upx` comprime o binary
- Apenas `ca-certificates` e `tzdata` no final

### Makefile

```makefile
# Definições de variáveis para ferramentas de análise
COVERAGE_DIR := coverage
GOCOV_VERSION := v1.0.0
JUNIT_REPORT_VERSION := v1.0.0
GOCOV_XML_VERSION := v1.1.0
GOLANGCI_LINT_VERSION := v1.61.0
GOSEC_VERSION := v2.21.4

SONAR_REPORT := sonar-report.json
UNIT_REPORT := unit-report.xml
COVERAGE_REPORT := coverage-report.xml

BIN_PATH := $(shell go env GOPATH)/bin

.PHONY: test-coverage setup-tools coverage-report help

help:
	@echo "Targets disponíveis:"
	@echo "  make setup-tools      - Instala ferramentas de análise"
	@echo "  make test-coverage    - Roda testes com cobertura"
	@echo "  make coverage-report  - Gera relatórios"

setup-tools:
	@echo "=== Instalando ferramentas de análise ==="
	go install github.com/axw/gocov/gocov@$(GOCOV_VERSION)
	go install github.com/jstemmer/go-junit-report@$(JUNIT_REPORT_VERSION)
	go install github.com/AlekSi/gocov-xml@$(GOCOV_XML_VERSION)
	curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | \
		sh -s -- -b $(BIN_PATH) $(GOLANGCI_LINT_VERSION)
	curl -sSfL https://raw.githubusercontent.com/securego/gosec/master/install.sh | \
		sh -s -- -b $(BIN_PATH) $(GOSEC_VERSION)
	@echo "✓ Ferramentas instaladas em $(BIN_PATH)"

test-coverage:
	@echo "=== Executando testes com cobertura ==="
	@mkdir -p $(COVERAGE_DIR)
	
	go test -v -json ./... \
		-covermode=count \
		-coverprofile=$(COVERAGE_DIR)/coverage.out \
		-coverpkg=$$(go list ./... | grep -v '/tests/' | paste -sd "," -) \
		| tee $(COVERAGE_DIR)/test-output.txt
	
	@echo "✓ Testes completados"

coverage-report:
	@echo "=== Gerando relatórios de cobertura ==="
	@mkdir -p $(COVERAGE_DIR)
	
	# Filtrar arquivos da cobertura (opcional)
	@echo "mode: count" > $(COVERAGE_DIR)/filtered_coverage.out
	@grep -h -v -E "^mode:|repository.go|database.go|models.*" \
		$(COVERAGE_DIR)/coverage.out >> $(COVERAGE_DIR)/filtered_coverage.out || true
	
	# Gerar JUnit report
	go-junit-report < $(COVERAGE_DIR)/test-output.txt > $(COVERAGE_DIR)/$(UNIT_REPORT)
	@echo "✓ JUnit report: $(COVERAGE_DIR)/$(UNIT_REPORT)"
	
	# Gerar HTML coverage
	go tool cover -html=$(COVERAGE_DIR)/filtered_coverage.out \
		-o $(COVERAGE_DIR)/coverage.html
	@echo "✓ HTML coverage: $(COVERAGE_DIR)/coverage.html"
	
	# Gerar JSON para SonarQube
	gocov convert $(COVERAGE_DIR)/coverage.out > $(COVERAGE_DIR)/demo-coverage.json
	gocov-xml < $(COVERAGE_DIR)/demo-coverage.json > $(COVERAGE_DIR)/$(COVERAGE_REPORT)
	@echo "✓ Coverage report: $(COVERAGE_DIR)/$(COVERAGE_REPORT)"

lint:
	@echo "=== Executando linter ==="
	golangci-lint run ./...

security:
	@echo "=== Verificando segurança ==="
	gosec ./...

build:
	@echo "=== Build da aplicação ==="
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/app ./cmd/main.go
	@echo "✓ Build concluído: bin/app"

run:
	@echo "=== Rodando aplicação ==="
	go run ./cmd/main.go

clean:
	@echo "=== Limpando ==="
	rm -rf bin/ $(COVERAGE_DIR)/
	go clean
	@echo "✓ Limpeza concluída"
```

**Targets principais:**
- `setup-tools` - Instala todas as ferramentas necessárias
- `test-coverage` - Roda testes e gera coverage
- `coverage-report` - Formata relatórios para SonarQube

### Exemplos de Relatórios

**coverage.out (saída do go test):**
```
mode: count
seu-servico/internal/handler/user.go:10.30,25.2 1 1
seu-servico/internal/service/user.go:5.32,18.2 1 1
seu-servico/internal/repository/user.go:8.35,20.2 0 0
```

**Coverage XML (para SonarQube):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<coverage version="1">
  <sources>
    <source>/app</source>
  </sources>
  <packages>
    <package name="internal/handler">
      <classes>
        <class name="user.go" filename="internal/handler/user.go">
          <lines>
            <line number="10" hits="1" branch="false"/>
            <line number="15" hits="1" branch="false"/>
          </lines>
        </class>
      </classes>
    </package>
  </packages>
</coverage>
```

### deployment.yaml (Prod)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seu-servico
  namespace: default
  labels:
    app: seu-servico
    env: prod
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: seu-servico
      env: prod
  template:
    metadata:
      labels:
        app: seu-servico
        env: prod
    spec:
      containers:
      - name: seu-servico
        image: YOUR_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/seu-servico-prod:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ENVIRONMENT
          value: "prod"
        - name: LOG_LEVEL
          value: "info"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### deployment.yaml (Stage)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: seu-servico-stage
  namespace: default
  labels:
    app: seu-servico
    env: stage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: seu-servico
      env: stage
  template:
    metadata:
      labels:
        app: seu-servico
        env: stage
    spec:
      containers:
      - name: seu-servico
        image: YOUR_ACCOUNT_ID.dkr.ecr.YOUR_REGION.amazonaws.com/seu-servico-stage:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ENVIRONMENT
          value: "stage"
        - name: LOG_LEVEL
          value: "debug"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
```

---

## 🎨 Frontend: Node.js + S3 + CloudFront

### Estrutura do Projeto

```
seu-servico-admin/
├── ci/
│   └── pipeline.yml
├── src/
│   ├── components/
│   ├── pages/
│   ├── services/
│   ├── App.vue
│   └── main.js
├── public/
├── dist/ (gerado no build)
├── package.json
├── vite.config.js (ou webpack.config.js)
├── .eslintrc.js
├── .gitignore
└── README.md
```

### package.json

```json
{
  "name": "seu-servico-admin",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build:admin": "vite build --outDir dist/admin",
    "build:widget": "vite build --config vite.widget.config.js --outDir dist/widget",
    "build": "npm run build:admin && npm run build:widget",
    "preview": "vite preview",
    "lint": "eslint --ext .js,.vue --ignore-path .gitignore src",
    "lint:fix": "eslint --ext .js,.vue --ignore-path .gitignore src --fix",
    "test": "vitest",
    "test:coverage": "vitest run --coverage"
  },
  "dependencies": {
    "vue": "^3.3.0",
    "axios": "^1.4.0",
    "pinia": "^2.1.0"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^4.0.0",
    "vite": "^4.3.0",
    "eslint": "^8.0.0",
    "eslint-plugin-vue": "^9.0.0",
    "vitest": "^0.33.0",
    "@vitest/ui": "^0.33.0",
    "@vitest/coverage-v8": "^0.33.0"
  }
}
```

**Detalhes importantes:**
- Scripts separados para diferentes builds (`build:admin`, `build:widget`)
- `npm run build` executa ambos
- `test:coverage` gera relatório de cobertura
- `lint` valida código antes do build

### vite.config.js

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  build: {
    outDir: 'dist/admin',
    sourcemap: false,
    minify: 'terser',
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['vue', 'axios', 'pinia']
        }
      }
    }
  },
  server: {
    port: 3000
  }
})
```

### .eslintrc.js

```javascript
module.exports = {
  root: true,
  env: {
    browser: true,
    node: true,
    es2021: true
  },
  extends: [
    'eslint:recommended',
    'plugin:vue/vue3-recommended'
  ],
  parserOptions: {
    ecmaVersion: 12,
    sourceType: 'module'
  },
  rules: {
    'vue/multi-word-component-names': 'off',
    'semi': ['error', 'never'],
    'quotes': ['error', 'single']
  }
}
```

### vitest.config.js (Testes)

```javascript
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html', 'xml'],
      exclude: ['node_modules/', 'tests/']
    }
  }
})
```

### Exemplo de Teste (src/components/Button.test.js)

```javascript
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import Button from './Button.vue'

describe('Button Component', () => {
  it('renders button text', () => {
    const wrapper = mount(Button, {
      props: {
        text: 'Click me'
      }
    })
    expect(wrapper.text()).toContain('Click me')
  })

  it('emits click event', async () => {
    const wrapper = mount(Button)
    await wrapper.trigger('click')
    expect(wrapper.emitted().click).toBeTruthy()
  })
})
```

### Build Output

Após `npm run build`:
```
dist/
├── admin/
│   ├── index.html
│   ├── assets/
│   │   ├── index-abc123.js
│   │   ├── vendor-def456.js
│   │   └── style-ghi789.css
│   └── ...
└── widget/
    ├── widget.html
    ├── assets/
    └── ...
```

---

## 📁 Estrutura do Repositório

### Padrão Backend (Go + Kubernetes)

```
seu-backend/
├── ci/
│   └── pipeline.yml                    # Trigger do pipeline
├── cd/
│   ├── prod/
│   │   └── deployment.yaml
│   └── stage/
│       └── deployment.yaml
├── cmd/
│   └── main.go
├── internal/
│   ├── handler/
│   ├── service/
│   ├── model/
│   └── repository/
├── pkg/
├── Dockerfile                          # Build multi-stage
├── Makefile                            # Targets: setup-tools, test-coverage, coverage-report
├── go.mod
├── go.sum
└── README.md
```

### Padrão Frontend (Node.js + S3)

```
seu-frontend/
├── ci/
│   └── pipeline.yml                    # Trigger do pipeline
├── src/
│   ├── components/
│   ├── pages/
│   ├── App.vue
│   └── main.js
├── public/
├── dist/ (gerado no build)
├── package.json                        # Scripts: build, lint, test:coverage
├── vite.config.js
├── .eslintrc.js
├── vitest.config.js
└── README.md
```

---

## 🔐 Service Connections

### Service Connection: Kubernetes (Prod)

**Tipo:** Kubernetes

| Campo | Valor |
|-------|-------|
| Connection name | `kubernetes-prod` |
| Server URL | `https://seu-cluster-prod.eks.amazonaws.com` |
| Kubeconfig | (conteúdo do ~/.kube/config) |

**Como obter:**
```bash
aws eks update-kubeconfig --region us-east-1 --name seu-cluster-prod
cat ~/.kube/config
```

### Service Connection: Kubernetes (Stage)

**Tipo:** Kubernetes

| Campo | Valor |
|-------|-------|
| Connection name | `kubernetes-stage` |
| Server URL | `https://seu-cluster-stage.eks.amazonaws.com` |
| Kubeconfig | (conteúdo do ~/.kube/config) |

---

## 🌐 Variáveis de Ambiente

### Variable Group: `variables`

**Localização:** `Pipelines` > `Library` > `Variable groups`

| Variável | Valor | Secret | Descrição |
|----------|-------|--------|-----------|
| `AWS_ACCOUNT_ID` | `123456789012` | ✓ | Account ID AWS |
| `AWS_REGION` | `us-east-1` | ✗ | Região AWS |
| `ECR_REPO_NAME_PROD` | `seu-servico-prod` | ✗ | Nome ECR produção |
| `ECR_REPO_NAME_STAGE` | `seu-servico-stage` | ✗ | Nome ECR staging |
| `S3_BUCKET_PROD` | `seu-frontend-prod` | ✗ | Bucket S3 produção |
| `S3_BUCKET_STAGE` | `seu-frontend-stage` | ✗ | Bucket S3 staging |
| `CLOUDFRONT_DIST_PROD` | `E1234ABCD5XYZ` | ✗ | CloudFront prod |
| `CLOUDFRONT_DIST_STAGE` | `E9876VWXYZ123` | ✗ | CloudFront stage |
| `SONAR_HOST_URL` | `https://sonar.empresa.com` | ✗ | URL SonarQube |
| `SONAR_TOKEN` | `squ_abc...` | ✓ | Token SonarQube |
| `SQUAD_NAME` | `seu-squad` | ✗ | Nome da squad |

---

## 📝 Pipeline de Configuração

### Pipeline Backend (Go + Kubernetes)

**Arquivo:** `ci/pipeline.yml`

```yaml
name: $(Build.BuildId)

trigger:
  branches:
    include:
      - stage
      - main
  paths:
    exclude:
      - cd/*
      - ci/*
      - Dockerfile

variables:
  - group: variables
  - name: appname
    value: 'seu-servico'
  - name: apppath
    value: '.'
  - name: Dockerfile
    value: 'Dockerfile'
  - name: microservices
    value: 'no'
  - name: monorepo
    value: 'no'
  - name: language
    value: 'go'
  - name: version
    value: '1.23.0'
  - name: testun
    value: 'yes'
  - name: infra
    value: 'kube'
  - name: image
    value: 'golang:1.23.0-alpine3.20,alpine:3.12.1'
  - name: squad
    value: '$(SQUAD_NAME)'

resources:
  repositories:
    - repository: pipeline
      type: git
      name: Joker/pipeline
      ref: refs/heads/main
      endpoint: azure-devops-indecx

stages:
  - template: kubernetes.yml@pipeline
```

### Pipeline Frontend (Node.js + S3)

**Arquivo:** `ci/pipeline.yml`

```yaml
name: $(Build.BuildId)

trigger:
  branches:
    include:
      - stage
      - main
  paths:
    exclude:
      - ci/*

variables:
  - group: variables
  - name: appname
    value: 'seu-frontend'
  - name: apppath
    value: '.'
  - name: language
    value: 'node'
  - name: version
    value: '18.18.0'
  - name: testun
    value: 'yes'
  - name: infra
    value: 'cloudfront'
  - name: squad
    value: '$(SQUAD_NAME)'

resources:
  repositories:
    - repository: pipeline
      type: git
      name: Joker/pipeline
      ref: refs/heads/main
      endpoint: azure-devops-indecx

stages:
  - template: cloudfront.yml@pipeline
```

---

## 🛠️ Agent - Pré-Requisitos

### Ferramentas Obrigatórias

#### 1. Docker
```bash
# Ubuntu
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER

# Teste
docker --version
docker run hello-world
```

#### 2. AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Teste
aws --version
```

#### 3. kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Teste
kubectl version --client
```

#### 4. Git
```bash
sudo apt-get install -y git

# Teste
git --version
```

#### 5. Go (Para projetos Go)
```bash
wget https://go.dev/dl/go1.23.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.23.0.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Teste
go version
```

#### 6. Node.js (Para projetos Node.js)
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install 18.18.0
nvm use 18.18.0

# Teste
node --version
npm --version
```

#### 7. SonarQube Scanner
```bash
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
unzip sonar-scanner-cli-5.0.1.3006-linux.zip
sudo mv sonar-scanner-5.0.1.3006-linux /opt/sonar-scanner
sudo ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/

# Teste
sonar-scanner --version
```

#### 8. Trivy
```bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.45.0/trivy_0.45.0_Linux-64bit.tar.gz
tar zxvf trivy_0.45.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/

# Teste
trivy --version
```

### Script de Instalação Completo

```bash
#!/bin/bash
set -e

echo "=== Atualizando sistema ==="
sudo apt-get update && sudo apt-get upgrade -y

echo "=== Docker ==="
sudo apt-get install -y docker.io
sudo usermod -aG docker $(whoami)

echo "=== AWS CLI ==="
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -o awscliv2.zip
sudo ./aws/install --update
rm -rf aws awscliv2.zip

echo "=== kubectl ==="
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

echo "=== Git ==="
sudo apt-get install -y git

echo "=== Go ==="
wget https://go.dev/dl/go1.23.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.23.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.profile
rm -f go1.23.0.linux-amd64.tar.gz

echo "=== Node.js (NVM) ==="
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm install 18.18.0
nvm alias default 18.18.0

echo "=== SonarQube Scanner ==="
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
unzip -o sonar-scanner-cli-5.0.1.3006-linux.zip
sudo mv sonar-scanner-5.0.1.3006-linux /opt/sonar-scanner
sudo ln -s /opt/sonar-scanner/bin/sonar-scanner /usr/local/bin/
rm -f sonar-scanner-cli-5.0.1.3006-linux.zip

echo "=== Trivy ==="
wget https://github.com/aquasecurity/trivy/releases/download/v0.45.0/trivy_0.45.0_Linux-64bit.tar.gz
tar zxvf trivy_0.45.0_Linux-64bit.tar.gz
sudo mv trivy /usr/local/bin/
rm -f trivy_0.45.0_Linux-64bit.tar.gz

echo ""
echo "=== ✅ Verificando instalações ==="
docker --version
aws --version
kubectl version --client
git --version
go version
node --version
npm --version
sonar-scanner --version
trivy --version

echo ""
echo "=== ✅ Instalação concluída! ==="
echo "Reinicie o shell para aplicar as mudanças: newgrp docker"
```

### Checklist de Ferramentas

| Ferramenta | Backend | Frontend | Versão |
|-----------|---------|----------|--------|
| Docker | ✓ | ✓ | 24.0+ |
| AWS CLI | ✓ | ✓ | 2.0+ |
| kubectl | ✓ | ✗ | 1.28+ |
| Git | ✓ | ✓ | 2.30+ |
| Go | ✓ | ✗ | 1.23+ |
| Node.js | ✗ | ✓ | 18.0+ |
| SonarQube Scanner | ✓ | ✓ | 5.0+ |
| Trivy | ✓ | ✓ | 0.45+ |

---

## 📋 Resumo: Backend vs Frontend

### Backend (Go + Kubernetes)
```
✓ Dockerfile multi-stage com upx
✓ Makefile com setup-tools, test-coverage, coverage-report
✓ deployment.yaml em cd/prod e cd/stage
✓ Pipeline dispara com: stage/main branches
✓ Executa: SAST → Build Docker → Push ECR → Deploy K8s
✓ Ferramentas: Go, Docker, kubectl, SonarQube, Trivy
```

### Frontend (Node.js + S3)
```
✓ package.json com: build, lint, test:coverage
✓ Múltiplos builds: build:admin, build:widget
✓ vitest para testes com cobertura
✓ Pipeline dispara com: stage/main branches
✓ Executa: SAST → Lint → Build → Push S3 → Invalidate CloudFront
✓ Ferramentas: Node.js, npm, SonarQube, Trivy
```

---