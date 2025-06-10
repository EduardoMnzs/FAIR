# Guia Completo: CRUD de Metadatas no FAIR Data Point

Este documento descreve o fluxo de trabalho completo para Criar, Ler, Atualizar e Deletar (CRUD) metadatas em uma instância do FAIR Data Point (FDP), com base na arquitetura configurada em nosso processo.

## 1. Pré-requisitos Essenciais

Antes de qualquer operação, garanta que seu ambiente esteja 100% correto:

### 1.1. Ambiente Docker Configurado

Seu arquivo `docker-compose.yml` deve estar configurado para expor a porta do servidor da API e injetar as variáveis de ambiente que definem o FDP raiz. Esta é a configuração final e correta:

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

**Ação:** Inicie o ambiente com `docker-compose up -d`. Todas as requisições de API devem ser feitas para `http://localhost:8080` ou `http://localhost`.

### 1.2. Token de Autenticação

Todas as operações de escrita (`POST`, `PUT`, `DELETE`) exigem um token.

-   **Endpoint:** `POST /tokens`
-   **Corpo:** `{"email": "seu-email", "password": "sua-senha"}`
-   **Uso:** Envie o token recebido no cabeçalho de todas as requisições seguintes: `Authorization: Bearer <seu_token_aqui>`.

----------

## 2. O Ciclo de Vida do Metadado

Todo metadado criado no FDP passa por dois estados:

-   **`DRAFT` (Rascunho):** O estado inicial após a criação. O metadado existe, mas não é considerado "oficial" e não pode ser usado como pai para outros recursos.
-   **`PUBLISHED` (Publicado):** O estado final. O metadado está "ao vivo" e pode ser referenciado por outros recursos.

----------

## 3. Operações CRUD de Metadatas

A criação de metadatas segue a hierarquia `FDP Raiz -> Catálogo -> Dataset -> Distribuição`.

### CREATE (Criar um Novo Metadado)

Esta é a operação mais complexa, pois define a estrutura e o parentesco.

-   **Endpoint:** `POST /{urlPrefix}` (ex: `/catalog`, `/dataset`, `/distribution`)
-   **Cabeçalhos Essenciais:** `Content-Type: text/turtle`
-   **Corpo (Body):** Um texto em formato Turtle (`.ttl`) que **obrigatoriamente** deve conter:
    1.  **O tipo do recurso:** Ex: `a dcat:Resource, dcat:Catalog ;`
    2.  **Todos os campos obrigatórios** definidos pelo schema (ex: `dcat:version`, `dct:publisher`).
    3.  **A relação de parentesco:** A propriedade `dct:isPartOf` apontando para a URL completa do recurso pai. (A única exceção é o FDP Raiz, que não tem pai).

#### Exemplo: Criando um Catálogo

Bash

```
# O corpo da requisição está no arquivo catalogo.ttl

# Conteúdo de catalogo.ttl:
# @prefix dcat: <http://www.w3.org/ns/dcat#> .
# @prefix dct: <http://purl.org/dc/terms/> .
# @prefix foaf: <http://xmlns.com/foaf/0.1/> .
#
# <> a dcat:Resource, dcat:Catalog ;
#     dct:title "Meu Catálogo" ;
#     dcat:version "1.0" ;
#     dct:publisher [ a foaf:Agent ; foaf:name "Meu Laboratório" ] ;
#     dct:isPartOf <http://localhost:8080> .

curl --location --request POST 'http://localhost:8080/catalog' \
--header 'Content-Type: text/turtle' \
--header 'Authorization: Bearer <seu_token_aqui>' \
--data @catalogo.ttl
```

-   **Resultado:** `201 Created`. O cabeçalho `Location` na resposta conterá a URL do novo catálogo.

----------

### READ (Ler um Metadado Existente)

Para ler os dados de um recurso específico, use seu UUID.

-   **Endpoint:** `GET /{urlPrefix}/{uuid}`
-   **Cabeçalhos:** `Accept: text/turtle`

#### Exemplo: Lendo um Catálogo

Bash

```
curl --location 'http://localhost:8080/catalog/uuid-do-catalogo-aqui' \
--header 'Accept: text/turtle'

```

-   **Resultado:** O corpo da resposta será os metadatas do catálogo em formato Turtle.

----------

### UPDATE (Atualizar um Metadado)

Existem dois tipos de atualização:

#### 3.1. Atualizar o Conteúdo

Para mudar o título, descrição, etc.

-   **Endpoint:** `PUT /{urlPrefix}/{uuid}`
-   **Corpo:** O **corpo inteiro e atualizado** do metadado em formato Turtle. Você precisa enviar todos os campos novamente, não apenas o que mudou.

#### Exemplo: Atualizando o título do Catálogo

Bash

```
# O corpo da requisição está no arquivo catalogo-atualizado.ttl
# (É o mesmo do create, mas com o dct:title modificado)

curl --location --request PUT 'http://localhost:8080/catalog/uuid-do-catalogo-aqui' \
--header 'Content-Type: text/turtle' \
--header 'Authorization: Bearer <seu_token_aqui>' \
--data @catalogo-atualizado.ttl

```

-   **Resultado:** `200 OK`.

#### 3.2. Atualizar o Estado (Publicar)

Para mover de `DRAFT` para `PUBLISHED`.

-   **Endpoint:** `PUT /{urlPrefix}/{uuid}/meta/state`
-   **Cabeçalhos:** `Content-Type: application/json`
-   **Corpo:** `{"current": "PUBLISHED"}`

#### Exemplo: Publicando o Catálogo

Bash

```
curl --location --request PUT 'http://localhost:8080/catalog/uuid-do-catalogo-aqui/meta/state' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <seu_token_aqui>' \
--data-raw '{ "current": "PUBLISHED" }'

```

-   **Resultado:** `200 OK`.

----------

### DELETE (Deletar um Metadado)

Para remover um recurso de metadatas.

-   **Endpoint:** `DELETE /{urlPrefix}/{uuid}`

#### Exemplo: Deletando o Catálogo

Bash

```
curl --location --request DELETE 'http://localhost:8080/catalog/uuid-do-catalogo-aqui' \
--header 'Authorization: Bearer <seu_token_aqui>'

```

-   **Resultado:** `204 No Content`.

----------

### Buscando Metadatas (SPARQL)

Para buscas complexas, use o endpoint `/search/query`.

-   **Endpoint:** `POST /search/query`
-   **Corpo:** Um JSON estruturado com as partes da sua query SPARQL.

#### Exemplo: Buscando por "Corais"

Bash

```
curl --location --request POST 'http://localhost:8080/search/query' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <seu_token_aqui>' \
--data-raw '{
    "graphPattern": "?entity ?relationPredicate ?relationObject . FILTER CONTAINS(LCASE(str(?relationObject)), LCASE(\"corais\"))"
}'

```

-   **Resultado:** Uma lista em JSON dos recursos que correspondem à busca.