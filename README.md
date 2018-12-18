# SonarQube Examplo de Configuração
-----------------------------------------------------------------------------------------------------------

#### Descrição: Exemplo de integração entre Gitlab CI e SonarQube em um projeto Maven.

Artigo: [Adicionando Code Analysis ao seu pipeline com SonarQube e GitLab CI](https://medium.com/@jeanmorais/adicionando-code-analysis-ao-seu-pipeline-com-sonarqube-e-gitlab-ci-1dab39aa3f75)

## Plugin 

[sonar-gitlab-plugin](https://github.com/gabrie-allaigre/sonar-gitlab-plugin)


## Roteiro

-----------------------------------------------------------------------------------------------------------

### Poc SonarQube com DotneCore
* Procedimento foi realizado usando o Play with Docker
* http://play-with-docker.com

### Subir SonarQube usando o Docker

* Exemplo abaixo não será usado, apenas para ilustrar, iremos usar o **Docker Compose** para facilitar a orquestração dos containers. 

```
docker run -d --name sonarqube -p 9000:9000 -p 9092:9092 -e SONARQUBE_JDBC_USERNAME=sonar -e SONARQUBE_JDBC_PASSWORD=sonar -e SONARQUBE_JDBC_URL=jdbc:postgresql://localhost/sonar sonarqube
```

### Sonar and Postgre

* link oficial do compose.

- https://github.com/SonarSource/docker-sonarqube/blob/master/recipes.md

* Criar o arquivo docker-compose.yml e copiar e colocar as linhas abaixo.

* Obs: O docker irá subir dois container, 1 com o Sonar e outro com banco de dados Postgre.

-----------------------------------------------------------------------------------------------------------

```
version: "2"

services:
  sonarqube:
    image: sonarqube
    ports:
      - "9000:9000"
    networks:
      - sonarnet
    environment:
      - SONARQUBE_JDBC_URL=jdbc:postgresql://db:5432/sonar
    volumes:
      - sonarqube_conf:/opt/sonarqube/conf
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions
      - sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins

  db:
    image: postgres
    networks:
      - sonarnet
    environment:
      - POSTGRES_USER=sonar
      - POSTGRES_PASSWORD=sonar
    volumes:
      - postgresql:/var/lib/postgresql
      # This needs explicit mapping due to https://github.com/docker-library/postgres/blob/4e48e3228a30763913ece952c611e5e9b95c8759/Dockerfile.template#L52
      - postgresql_data:/var/lib/postgresql/data

networks:
  sonarnet:
    driver: bridge

volumes:
  sonarqube_conf:
  sonarqube_data:
  sonarqube_extensions:
  sonarqube_bundled-plugins:
  postgresql:
  postgresql_data:
```

### Rode o comando do Docker Compose

```
docker-compose up -d
```
* Esse comando irá baixar as imagens do Docker e Postgre

### Exemplo de configuração

* Executar esses passos no **SonarQube** que foi iniciado com o comando docker-compo up -d na porta 9000.

* Login e senha: admin

```
  Sonar Token: 6e284eab95abb82bf955663a2f4a2f8445fb51c3
```

* Scrip que o Sonar gera para ser configurado no Pipeline

```
  mvn sonar:sonar \
  -Dsonar.host.url=http://ip172-18-0-3-bf4lsfk9cs9g008ekns0-9000.direct.labs.play-with-docker.com \
  -Dsonar.login=6e284eab95abb82bf955663a2f4a2f8445fb51c3
```

* A Url e o Token do Sonar são passados via variavel. 

#### Exemplo no Maven
```
      - mvn --batch-mode verify sonar:sonar -Drevision=$REVISION_STABLE -Dsonar.host.url=$SONAR_URL -Dsonar.login=$SONAR_LOGIN -Dsonar.gitlab.project_id=$CI_PROJECT_PATH -Dsonar.gitlab.ref_name=$CI_COMMIT_REF_NAME -Dsonar.gitlab.commit_sha=$CI_COMMIT_SHA -Dsonar.gitlab.failure_notification_mode=exit-code -Dsonar.analysis.mode=publish
```

#### Exemplo no DotnetCore

```
      - dotnet sonarscanner begin /k:"$SONAR_PROJECT_KEY" /d:sonar.host.url="$SONAR_URL" /d:sonar.login="$SONAR_TOKEN"
```

### Configurar as Variaveis para o pipeline no Gitlab

* https://gitlab.com/cristianvitortrucco/sonarpoc/settings/ci_cd

```
SONAR_LOGIN 6e284eab95abb82bf955663a2f4a2f8445fb51c3

SONAR_URL 	http://ip172-18-0-3-bf4lsfk9cs9g008ekns0-9000.direct.labs.play-with-docker.com
```

### Configuração do Arquivo .gitlab.yaml

* Obs: o arquivo deve estar na raiz do repositório

```
image: microsoft/dotnet:latest

stages:
  - test
  - code_analysis
  - deploy

release:
  stage: deploy
  only:
    - master
  artifacts:
    paths:
      - publish/
  script:
    # The output path is relative to the position of the csproj-file
     - dotnet publish -c Release -o  MyProject/MyProject.csproj


test:
  stage: test
  variables:
    test_path: "teste-api/"
  script:
    - "cd $test_path"
    - "dotnet build"
    - "dotnet test"
  only:
      - develop

sonar:
  stage: code_analysis
  variables: 
    SONAR_SCANNER_OPTS: "-Xmx1024m"
  script: 
    - export PATH="$PATH:/root/.dotnet/tools"
    - export SONAR_SCANNER_OPTS
    - apt-get update
    - apt-get install openjdk-8-jre -y
    - dotnet tool install --global dotnet-sonarscanner --version 4.4.2
    
    #- dotnet tool install --global coverlet.console
    #- dotnet test ./ --configuration Debug --output ../../output-tests  /p:CollectCoverage=true /p:CoverletOutputFormat=opencover /p:CoverletOutput='./output-coverage/coverage.opencover.xml' /p:ExcludeByFile="tests/**"
    
    - dotnet sonarscanner begin /k:"$SONAR_PROJECT_KEY" /d:sonar.host.url="$SONAR_URL" /d:sonar.login="$SONAR_TOKEN"
    - dotnet build
    - dotnet sonarscanner end /d:sonar.login="$SONAR_TOKEN"
  only:
    - develop

```


#### Referências para Poc.

* Usando exemplo: https://medium.com/@jeanmorais/adicionando-code-analysis-ao-seu-pipeline-com-sonarqube-e-gitlab-ci-1dab39aa3f75
* Repo: https://github.com/jeanmorais/sonar-maven-ci-example/blob/master/.gitlab-ci.yml
* Doc Docker Sonar: https://docs.docker.com/samples/library/sonarqube/
