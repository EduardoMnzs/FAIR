
# Guia Rápido: Iniciando o FDP com seu `docker-compose.yml`

Este guia mostra como iniciar uma instância do FAIR Data Point (FDP) usando a sua configuração específica do Docker Compose.

## Pré-requisitos

Garanta que você tenha os seguintes softwares instalados:

-   **Docker**: [Instrução de Instalação do Docker](https://docs.docker.com/engine/install/)
-   **Docker Compose**: [Instrução de Instalação do Docker Compose](https://docs.docker.com/compose/install/)

## Passo 1: Crie o Arquivo de Configuração

1.  Crie um novo diretório para o seu projeto e entre nele:
    
    Bash
    
    ```
    mkdir meu-fdp-simples
    cd meu-fdp-simples
    
    ```
    
2.  Dentro deste diretório, crie um arquivo chamado `docker-compose.yml` e cole o conteúdo exato que você forneceu:
    
    YAML
    
    ```
    version: '3.8'

    services:
      fdp:
        image: fairdata/fairdatapoint:1.17
        depends_on:
          - mongo
        ports:
          - "8080:80"
        environment:
          - FDP_INSTANCE_CLIENTURL=http://localhost/
          - FDP_INSTANCE_PERSISTENTURL=http://localhost:8080/
          - FDP_METADATA_REPOSITORY_TITLE=Meu FAIR Data Point
          - FDP_METADATA_PUBLISHER_NAME=Minha Organização

      fdp-client:
        image: fairdata/fairdatapoint-client:latest
        ports:
          - "80:80"
        depends_on:
          - fdp
        environment:
          - FDP_HOST=fdp

      mongo:
        image: mongo:4.0.12
    
    ```
    

> ### ⚠️ Atenção: Implicações desta Configuração
> 
> Este arquivo `docker-compose.yml` é funcional, mas é importante entender duas coisas sobre ele:
> 
> 1.  **Credenciais Padrão**: Como não definimos um usuário e senha, a aplicação usará o administrador padrão. Você precisará usar as credenciais abaixo para o primeiro login.
> 2.  **Sem Persistência de Dados**: Esta configuração não cria um "volume" para o banco de dados. Isso significa que **todos os metadados que você criar serão permanentemente apagados** sempre que você parar e remover os containers com o comando `docker-compose down`.

## Passo 2: Inicie a Aplicação

Abra um terminal no diretório onde você salvou o arquivo e execute:

Bash

```
docker-compose up -d

```

Este comando irá baixar as imagens (se for a primeira vez) e iniciar os três serviços em segundo plano.

## Passo 3: Verifique e Acesse o FDP

1.  **Verifique se os Containers estão em Execução:** Use o comando `docker-compose ps` para ver o status dos seus serviços. Todos devem estar com o estado `Up` ou `running`.
    
2.  **Acesse a Interface Web:** Abra seu navegador e vá para o endereço: **[http://localhost](https://www.google.com/search?q=http://localhost)**
    
3.  **Faça Login com o Usuário Padrão:** A tela de login será exibida. Use as seguintes credenciais padrão para entrar:
    
    -   **Usuário:** `albert.einstein@example.com`
    -   **Senha:** `password`

Após o login, você terá acesso ao painel de administração do FAIR Data Point e poderá começar a adicionar catálogos e datasets.

## Passo 4: Como Parar a Aplicação

Quando quiser parar os containers, use o seguinte comando no mesmo diretório:

Bash

```
docker-compose down

```

> **Lembrete Importante:** Conforme mencionado, executar `docker-compose down` com esta configuração **apagará todos os dados** que você inseriu no FDP. Esta configuração é ideal para testes rápidos e demonstrações, mas não para armazenar metadados de forma permanente.