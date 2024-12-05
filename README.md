# Projeto DevOps  
Este projeto apresenta um ambiente completo de DevOps, integrando uma aplicação web desenvolvida com Flask e um banco de dados MariaDB. A solução abrange uma pipeline de CI/CD automatizada no Jenkins e monitoramento em tempo real com Prometheus e Grafana, entregando um ciclo de desenvolvimento eficiente, automatizado e com alta observabilidade.

---

### Objetivo  
Demonstrar a implementação de um fluxo completo de integração e entrega contínuas (CI/CD) com as seguintes funcionalidades: 
- Automação de testes, build e deploy utilizando Jenkins;
- Monitoramento detalhado de métricas da aplicação e do banco de dados com Prometheus e Grafana;
- Criação de um ambiente confiável, escalável e observável para aplicações modernas.

---

### Estrutura do Projeto  
O projeto está estruturado conforme o diagrama abaixo, destacando os principais diretórios e arquivos:

```plaintext
├── docker-compose.yml
├── exporter
│   └── mysqld_exporter.env
├── flask
│   ├── app.py
│   ├── Dockerfile_flask
│   ├── requirements.txt
│   └── test_flask_app.py
├── grafana
│   ├── dashboards
│   │   └── mariadb_dashboard.json
│   ├── Dockerfile_grafana
│   └── provisioning
│       ├── dashboard.yml
│       └── datasource.yml
├── Jenkinsfile
├── mariadb
│   └── Dockerfile_mariadb
└── prometheus
    └── prometheus.yml
```  

---

### Detalhe dos Containers  

### 1. **Flask**  
- **Função:** Aplicação web principal.  
- **Arquivos:**  
  - `app.py`: Código da aplicação com endpoints básicos.  
  - `Dockerfile_flask`: Configuração da imagem Docker do serviço Flask.  
  - `test_flask_app.py`: Testes unitário.  

### 2. **MariaDB**  
- **Função:** Banco de dados relacional usado pela aplicação Flask.  
- **Arquivos:**  
  - `Dockerfile_mariadb`: Configuração da imagem Docker do MariaDB.  

### 3. **Prometheus**  
- **Função:** Coleta e armazena métricas do ambiente.  
- **Arquivos:**  
  - `prometheus.yml`: Configura os alvos de monitoramento, como Flask e MariaDB.  

### 4. **Grafana**  
- **Função:** Visualização e análise de métricas coletadas pelo Prometheus.  
- **Arquivos:**  
  - `mariadb_dashboard.json`: Dashboard para monitorar o MariaDB.  
  - `datasource.yml`: Conexão com o Prometheus como fonte de dados.  

### 5. **MySQL Exporter**  
- **Função:** Exporta métricas do MariaDB para o Prometheus.  
- **Arquivo:**  
  - `mysqld_exporter.env`: Configuração do MySQL Exporter.  

### 6. **Jenkins**  
- **Função:** Automação do pipeline CI/CD.  
- **Arquivo:**  
  - `Jenkinsfile`: Define etapas como clonagem do repositório, build, deploy e testes.  


---

### **Configurações dos Serviços**

### **1. Banco de Dados: MariaDB**
Arquivo: `Dockerfile_mariadb`  
```dockerfile
# Dockerfile para MariaDB
# Salve este arquivo como Dockerfile.mariadb na raiz do projeto

# Dockerfile (MariaDB)
FROM mariadb:10.5

# Defina as variáveis de ambiente para o banco de dados
ENV MYSQL_ROOT_PASSWORD=root_password
ENV MYSQL_DATABASE=school_db
ENV MYSQL_USER=flask_user
ENV MYSQL_PASSWORD=flask_password

EXPOSE 3306
```
---

### **2. Backend: Aplicação Flask**

Arquivo: `app.py`  
```python
# Código principal do Flask (app.py)
import time
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_appbuilder import AppBuilder, SQLA
from flask_appbuilder.models.sqla.interface import SQLAInterface
from flask_appbuilder import ModelView
from sqlalchemy.exc import OperationalError
from prometheus_flask_exporter import PrometheusMetrics
import logging

app = Flask(__name__)

metrics = PrometheusMetrics(app)
# Configuração da chave secreta para sessões
app.config['SECRET_KEY'] = 'minha_chave_secreta_super_secreta'  # Substitua por uma chave segura

# Configuração do banco de dados
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:root_password@mariadb/school_db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

# Inicializar o banco de dados e o AppBuilder
db = SQLAlchemy(app)
appbuilder = AppBuilder(app, db.session)

    
# Configuração do log
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Modelo de Aluno - Definição da tabela 'Aluno' no banco de dados
class Aluno(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    nome = db.Column(db.String(50), nullable=False)
    sobrenome = db.Column(db.String(50), nullable=False)
    turma = db.Column(db.String(50), nullable=False)
    disciplinas = db.Column(db.String(200), nullable=False)
    matricula = db.Column(db.String(20), nullable=False)
    
# Tentar conectar até o MariaDB estar pronto
attempts = 5
for i in range(attempts):
    try:
        with app.app_context():
            db.create_all()  # Inicializa o banco de dados
            # Criar um usuário administrador padrão
            if not appbuilder.sm.find_user(username='admin'):
                appbuilder.sm.add_user(
                    username='admin',
                    first_name='Admin',
                    last_name='User',
                    email='admin@admin.com',
                    role=appbuilder.sm.find_role(appbuilder.sm.auth_role_admin),
                    password='admin'
                )
        logger.info("Banco de dados inicializado com sucesso.")
        break
    except OperationalError:
        if i < attempts - 1:
            logger.warning("Tentativa de conexão com o banco de dados falhou. Tentando novamente em 5 segundos...")
            time.sleep(5)  # Aguarda 5 segundos antes de tentar novamente
        else:
            logger.error("Não foi possível conectar ao banco de dados após várias tentativas.")
            raise


# Visão do modelo Aluno para o painel administrativo
class AlunoModelView(ModelView):
    datamodel = SQLAInterface(Aluno)
    list_columns = ['id', 'nome', 'sobrenome', 'turma', 'disciplinas', 'matricula']

# Adicionar a visão do modelo ao AppBuilder
appbuilder.add_view(
    AlunoModelView,
    "Lista de Alunos",
    icon="fa-folder-open-o",
    category="Alunos",
)

# Rota para listar todos os alunos - Método GET
@app.route('/alunos', methods=['GET'])
def listar_alunos():
    alunos = Aluno.query.all()
    output = [{'id': aluno.id, 'nome': aluno.nome, 'sobrenome': aluno.sobrenome, 'turma': aluno.turma, 'disciplinas': aluno.disciplinas, 'matricula': aluno.matricula} for aluno in alunos]
    return jsonify(output)

# Rota para adicionar um aluno - Método POST
@app.route('/alunos', methods=['POST'])
def adicionar_aluno():
    data = request.get_json()
    novo_aluno = Aluno(nome=data['nome'], sobrenome=data['sobrenome'], turma=data['turma'], disciplinas=data['disciplinas'], matricula=data['matricula'])
    db.session.add(novo_aluno)
    db.session.commit()
    logger.info(f"Aluno {data['nome']} {data['sobrenome']} adicionado com sucesso!")
    return jsonify({'message': 'Aluno adicionado com sucesso!'}), 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

Arquivo: `requirements.txt`  
```plaintext
Flask==1.1.4  # Versão compatível com Flask-AppBuilder
Flask-SQLAlchemy==2.4.4  # Extensão do Flask para integração com SQLAlchemy
PyMySQL==0.9.3  # Biblioteca para conexão do Python com o banco de dados MariaDB
Flask-AppBuilder==3.3.0  # Versão compatível com Flask 1.x
Werkzeug==1.0.1  # Versão compatível do Werkzeug para evitar erros de importação
MarkupSafe==2.0.1  # Versão compatível com Jinja2 e Flask
WTForms==2.3.3  # Versão compatível com Flask-AppBuilder que contém o módulo 'compat'
prometheus-flask-exporter==0.18.3
pytest==6.2.5
pytest-flask==1.2.0
Flask-Testing==0.8.0
```

Arquivo: `Dockerfile_flask`  
```dockerfile
# Dockerfile (Flask AppBuilder)
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

#COPY . .
COPY app.py /app/

COPY test_flask_app.py /app/

CMD ["flask", "run", "--host=0.0.0.0"]
```

Arquivo: `test_app.py`
````
import pytest
from flask import Flask
from flask.testing import FlaskClient

# Importar a aplicação Flask
from app import app  # Assumindo que seu arquivo principal é app.py

@pytest.fixture
def client():
    with app.test_client() as client:
        yield client

def test_listar_alunos(client: FlaskClient):
    """Testa a rota GET /alunos"""
    response = client.get('/alunos')
    assert response.status_code == 200
    assert isinstance(response.json, list)

def test_adicionar_aluno(client: FlaskClient):
    """Testa a rota POST /alunos"""
    new_aluno = {
        "nome": "Gustavo",
        "sobrenome": "Zeglan",
        "turma": "4 periodo",
        "disciplinas": "DevOps",
        "matricula": "2023"
    }
    response = client.post('/alunos', json=new_aluno)
    assert response.status_code == 201
    assert response.json['message'] == 'Aluno adicionado com sucesso!'
````
---

### **3. Monitoramento: Prometheus e Grafana**
Arquivo: `prometheus.yml`  
```yaml
global:
  scrape_interval: 15s  # Intervalo padrão de coleta de métricas para todos os jobs

scrape_configs:
  - job_name: 'prometheus'  # Nome do job para o Prometheus
    static_configs:
      - targets: ['localhost:9090']  # Definição do alvo, neste caso o próprio Prometheus

  - job_name: 'mysqld_exporter'  # Nome do job para o MySQL Exporter
    static_configs:
      - targets: ['mysqld_exporter:9104']  # Definição do alvo para coletar métricas do MySQL Exporter

```

Arquivo: `datasource.yml`  
```yaml
apiVersion: 1  # Versão da API utilizada para configuração do Grafana

datasources:
  - name: Prometheus  # Nome da fonte de dados que será adicionada no Grafana
    type: prometheus  # Tipo da fonte de dados, neste caso, Prometheus
    access: proxy  # Tipo de acesso, 'proxy' significa que as requisições passarão pelo backend do Grafana
    url: http://prometheus:9090  # URL para acessar o Prometheus dentro da rede do Docker
    isDefault: true  # Define esta fonte de dados como a padrão no Grafana
    jsonData:
      timeInterval: 5s  # Intervalo de tempo padrão para coleta de dados
```

Arquivo: `dashboard.yml`  
```yaml
apiVersion: 1

providers:
  - name: 'MariaDB Dashboards'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
````

Arquivo: `Dockerfile_grafana`
````
FROM grafana/grafana:latest

USER root

# Criar os diretórios necessários
RUN mkdir /var/lib/grafana/dashboards

# Copiar os arquivos de provisionamento e dashboards
COPY provisioning/datasource.yml /etc/grafana/provisioning/datasources/
COPY provisioning/dashboard.yml /etc/grafana/provisioning/dashboards/
COPY dashboards/mariadb_dashboard.json /var/lib/grafana/dashboards/

# Ajustar permissões para o usuário grafana
RUN chown -R 472:472 /etc/grafana/provisioning

USER grafana
````

Arquivo: `mariadb_dashboard.json`
````
{
  "uid": "simple_dashboard",
  "title": "Desempenho HTTP - Métricas Padrão",
  "tags": ["HTTP", "Prometheus", "Desempenho"],
  "timezone": "browser",
  "schemaVersion": 16,
  "version": 1,
  "panels": [
    {
      "type": "graph",
      "title": "Taxa de Requisições HTTP",
      "datasource": "Prometheus",
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 6 },
      "targets": [
        {
          "expr": "rate(prometheus_http_requests_total[1m])",
          "legendFormat": "Requisições/s",
          "refId": "A"
        }
      ],
      "lines": true,
      "linewidth": 2,
      "fill": 1
    },
    {
      "type": "graph",
      "title": "Latência Média de Requisições",
      "datasource": "Prometheus",
      "gridPos": { "x": 12, "y": 0, "w": 12, "h": 6 },
      "targets": [
        {
          "expr": "rate(prometheus_http_request_duration_seconds_sum[1m]) / rate(prometheus_http_request_duration_seconds_count[1m])",
          "legendFormat": "Latência Média (s)",
          "refId": "B"
        }
      ],
      "lines": true,
      "linewidth": 2,
      "fill": 1
    },
    {
      "type": "graph",
      "title": "Requisições HTTP em Andamento",
      "datasource": "Prometheus",
      "gridPos": { "x": 0, "y": 6, "w": 12, "h": 6 },
      "targets": [
        {
          "expr": "promhttp_metric_handler_requests_in_flight",
          "legendFormat": "Em andamento",
          "refId": "C"
        }
      ],
      "lines": true,
      "linewidth": 2,
      "fill": 1
    },
    {
      "type": "graph",
      "title": "Requisições HTTP - Taxa de Erro",
      "datasource": "Prometheus",
      "gridPos": { "x": 12, "y": 6, "w": 12, "h": 6 },
      "targets": [
        {
          "expr": "rate(prometheus_http_requests_total{status!~'2..'}[1m])",
          "legendFormat": "Taxa de Erro",
          "refId": "D"
        }
      ],
      "lines": true,
      "linewidth": 2,
      "fill": 1
    }
  ],
  "refresh": "10s",
  "time": { "from": "now-1h", "to": "now" },
  "overwrite": true
}
````
---

Arquivo: `Jenkinsfile`
````
pipeline {
    agent any

    environment {
        REPOSITORY = 'https://github.com/GustavoZeglan/trabalho-devops-2391902.git'
        BRANCH = 'main'
        CREDENTIAL = 'jenkins-user-github'
    }

    stages {
        stage('Clone the Git repository') {
            steps {
                git branch: "${BRANCH}", credentialsId: "${CREDENTIAL}", url: "${REPOSITORY}"
            }
        }

        stage('Start the containers') {
            steps {
                script {
                    sh '''
                        docker compose build 
                    '''

                    sh '''
                        docker compose up -d
                    '''
                }
            }
        }

        stage('Make the tests') {
            steps {
                script {
                    sh 'sleep 40' 
                    sh 'docker compose run --rm test'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}
````
---

### **Configuração Docker Compose**
Arquivo: `docker-compose.yml`
````

services:
  mariadb:
    container_name: mariadb
    build:
      context: ./mariadb
      dockerfile: Dockerfile_mariadb
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: school_db
      MYSQL_USER: flask_user
      MYSQL_PASSWORD: flask_password
    networks:
      - app_network

  flask:
    container_name: flask_app
    build:
      context: ./flask
      dockerfile: Dockerfile_flask
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=development
      - SQLALCHEMY_DATABASE_URI=mysql+pymysql://flask_user:flask_password@mariadb:3306/school_db
    depends_on:
      - mariadb
    networks:
      - app_network

  prometheus:
    container_name: prometheus
    image: prom/prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    depends_on:
      - mysqld_exporter
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    networks:
      - app_network

  mysqld_exporter:
    container_name: mysqld_exporter
    image: prom/mysqld-exporter
    ports:
      - "9104:9104"
    environment:
      DATA_SOURCE_NAME: "user:password@(mariadb:3306)/"
    depends_on:
      - mariadb
    networks:
      - app_network

  grafana:
    container_name: grafana
    build:
      context: ./grafana
      dockerfile: Dockerfile_grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - app_network

  test:
    container_name: test_flask_app
    build:
      context: ./flask
      dockerfile: Dockerfile_flask
    command: ["pytest", "/app/test_flask_app.py"]
    depends_on:
      - mariadb
      - flask
    environment:
      - DATABASE_URL=mysql+pymysql://root:root_password@mariadb/school_db
    networks:
      - app_network

networks:
  app_network:
    driver: bridge
````

---

### **Passo a Passo de Execução e Resultados**
Abaixo estão as imagens que ilustram os resultados do projeto em execução:

- **Jenkins**  
Acesse a URL: `http://localhost:8080`.  

1. **Faça login no Jenkins** com suas credenciais.  

2. **Crie um Novo Projeto**:  
   - Clique em "Nova Tarefa".  
   - Insira um nome para o pipeline, como **"trabalho_devops"**.  
   - Escolha a opção **"Pipeline"** e clique em **"Tudo certo"**.  

![Imagem Jenkins](image/jenkins1.png)  

3. **Configurar o Pipeline**:  
   - Na página de configuração do pipeline, vá até a seção **"Build Triggers"**.  
   - Ative a opção **"Consultar periodicamente o SCM"**.  
   - Adicione o seguinte cronograma para que o Jenkins verifique o repositório regularmente: `H/5 * * * *`.  
![Imagem Jenkins](image/jenkins2.png)  

1. **Definir o Script do Pipeline**:  
   - Role até a seção **"Pipeline"**.  
   - Selecione **"Pipeline Script from SCM"**.  
![Imagem Jenkins](image/jenkins3.png)  

1. **Configurar o Repositório Git**:  
   - No campo **SCM**, escolha **"Git"**.  
   - Informe o endereço do repositório do projeto:  
     `https://github.com/GustavoZeglan/trabalho-devops-2391902.git`.
   - Informe a branch na qual irá acessar os dados. 

2. **Salvar as Alterações**:  
   - Clique em **Salvar** para finalizar a configuração.  

3. **Executar o Pipeline**  
   - Acesse a página inicial do Jenkins e localize o pipeline recém-criado.

   - Clique no pipeline e, em seguida, selecione a opção **"Construir Agora"** para dar início à execução.
         
   ![Imagem Jenkins](image/jenkins4.png)

- **Grafana**:
Acesse a URL: `http://localhost:3000`

Faça login no Grafana:

- Usuário: admin
- Senha: admin 

Verifique a dashboard provisionado automaticamente e visualize as métricas de requisições.

![Dashboard](image/grafana.png)