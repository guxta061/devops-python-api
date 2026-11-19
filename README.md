devops-python-api

API de tarefas em Python (FastAPI) usada para demonstrar uma esteira completa de CI/CD com GitHub Actions, Docker, Amazon ECR e Amazon EC2.

Trabalho da disciplina Integração DevOps.

1. Descrição da aplicação

API didática que expõe endpoints para gerenciar tarefas e um endpoint de saúde (/health) que informa o status e a versão da aplicação. O objetivo do projeto é demonstrar que qualquer alteração aprovada na branch main é testada, empacotada, publicada e implantada automaticamente.

2. Tecnologias utilizadas
Python 3.13 e FastAPI
Uvicorn (servidor ASGI)
Pytest e pytest-cov (testes e cobertura)
Ruff (análise estática e formatação)
Docker
GitHub e GitHub Actions
Amazon ECR (registro privado de imagens)
Amazon EC2 (Ubuntu) com IAM Role para acesso ao ECR
3. Execução local
bash
git clone https://github.com/guxta061/devops-python-api.git
cd devops-python-api

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt

uvicorn app.main:app --reload

A API fica disponível em http://127.0.0.1:8000 e a documentação interativa em http://127.0.0.1:8000/docs.

Para rodar os testes:

bash
pytest
4. Docker

Construir a imagem:

bash
docker build -t devops-python-api .

Executar o contêiner:

bash
docker run -d --name devops-python-api -p 8000:8000 devops-python-api

Verificar:

bash
curl http://127.0.0.1:8000/health

Resposta esperada: {"status":"ok","version":"1.1.0"}

5. Integração contínua (CI)

Arquivo: .github/workflows/ci.yml

Executado na abertura ou atualização de um Pull Request para a main (e manualmente, via workflow_dispatch).

Jobs:

Qualidade e testes
baixa o código
configura o Python (versão do arquivo .python-version)
instala as dependências (requirements-dev.txt)
ruff check e ruff format --check
executa pytest com cobertura e gera coverage.xml e reports/junit.xml
publica os relatórios como artefato
Build e teste do container (só roda se o job anterior passar)
constrói a imagem Docker
sobe o contêiner e valida o endpoint /health

O merge só deve ser feito com os dois checks verdes.

6. Entrega contínua (CD)

Arquivo: .github/workflows/deploy.yml

Executado automaticamente a cada push na main (ou seja, após o merge de um Pull Request).

Jobs, em sequência (cada um depende do anterior):

Testes automatizados: executa pytest. Se falhar, nada é publicado.
Build e push no ECR
autentica na AWS com credenciais guardadas no GitHub (secrets do environment DES)
faz login no Amazon ECR
constrói a imagem e publica com a tag igual ao hash do commit (github.sha)
Deploy na EC2
conecta por SSH na instância (chave privada guardada em secret)
autentica o Docker da instância no ECR usando a IAM Role da EC2
baixa a imagem do commit (docker pull)
remove o contêiner anterior e inicia o novo (docker run, porta 8000)
verifica o /health e lista o contêiner em execução

Nenhum comando é executado manualmente na EC2 durante o deploy.

Configuração no GitHub (sem expor credenciais)

Environment DES:

Tipo	Nome
Secret	AWS_ACCESS_KEY_ID
Secret	AWS_SECRET_ACCESS_KEY
Secret	AWS_SESSION_TOKEN
Secret	ECR_REPOSITORY
Secret	EC2_USER
Secret	EC2_SSH_PRIVATE_KEY
Variable	EC2_HOST

Nenhuma credencial é versionada no repositório.

7. Arquitetura de implantação
Desenvolvedor
     |
     |  branch + Pull Request
     v
 GitHub  --(PR)-->  CI: testes + build do container
     |
     |  merge na main
     v
 GitHub Actions (deploy.yml)
     |  1. testes
     |  2. build da imagem
     |  3. push (tag = hash do commit)
     v
 Amazon ECR (repositório privado)
     |
     |  4. SSH
     v
 Amazon EC2 (Ubuntu + Docker, IAM Role com leitura no ECR)
     |  5. docker pull
     |  6. docker run -p 8000:8000
     v
 API acessível em http://<IP-PUBLICO-DA-EC2>:8000

Configuração da EC2:

Ubuntu com Docker instalado e em execução
AWS CLI instalado
IAM Role (LabRole, via instance profile) com permissão de leitura no ECR
Security group com entrada nas portas 22 (SSH) e 8000 (API)
8. URL de acesso à API
http://<IP-PUBLICO-DA-EC2>:8000/health

O IP público da EC2 muda quando a instância é parada e iniciada. IP usado na demonstração: 13.222.130.146 (preencher/atualizar conforme necessário).

9. Como verificar a versão implantada

Pelo navegador ou terminal:

bash
curl http://<IP-PUBLICO-DA-EC2>:8000/health

O campo version mostra a versão em execução. Também é possível conferir qual imagem está rodando na EC2:

bash
docker ps

A coluna IMAGE mostra a tag, que corresponde ao hash do commit publicado no ECR.

10. Evidências

Prints salvos na pasta docs/evidencias/ (ou anexados na entrega):

CI executado com sucesso no Pull Request
Falha do CI barrando uma alteração com teste desatualizado
Pipeline de deploy com os três jobs verdes
Imagem no Amazon ECR com as tags dos commits
docker ps na EC2 com o contêiner em execução
API respondendo version: 1.0.0 (primeiro deploy)
API respondendo version: 1.1.0 (segundo deploy, automático)
Demonstração de atualização automática
Deploy	Commit	Versão
1º	b634349	1.0.0
2º	810b0ff	1.1.0

O segundo deploy ocorreu automaticamente após o merge na main, sem acesso manual à EC2.

11. Pull Requests
PR 1 (CI/CD inicial): <link>
PR 2 (correção do deploy): <link>
PR 3 (versão 1.1.0): <link>
12. Estrutura do repositório
devops-python-api/
├── .github/workflows/
│   ├── ci.yml
│   └── deploy.yml
├── app/
├── tests/
├── docs/
├── Dockerfile
├── .dockerignore
├── requirements-dev.txt
├── pyproject.toml
└── README.md
