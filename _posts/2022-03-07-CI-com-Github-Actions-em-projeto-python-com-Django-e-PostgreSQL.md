---
layout: post
title: CI com Github Actions em projeto python com Django e PostgreSQL
img_path: /assets/img/
image: /assets/img/github-actions-thumbnail.png
categories: [DevOps, Github Actions]
tags: [github actions, devops, ci, integração contínua, python, django, postresql]
---

{% raw %}
Tenho dedicado um tempo a me familiarizar com a forma de construir pipelines de integração contínua com o github actions e, depois de algumas tentativas de construir uma para um projeto Django + PostgreSQL, resolvi documentar.

## Um primeiro arquivo do CI

O github não quer ver a gente fazendo esforço e trata logo de sugerir alguns modelos de CI muito úteis com base nos arquivos do projeto. Por exemplo, abaixo alguns modelos e o que escolhi pra configurar um primeiro workflow considerando apenas o django, com testes e linting.

![Github Actions sugere alguns modelos de CI para o seu projeto](github-actions-ci-suggestion.png)

Em relação ao modelo sugerido alterei apenas a última linha para adequar à estrutura de pastas do projeto e o array em python-version. O fluxo todo usa apenas a máquina principal, sem container de serviço, e ficou assim:

```yaml
name: Django CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.7]

    steps:
    - uses: actions/checkout@v2
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run Tests
      run: |
        python app/manage.py test app/ && flake8 --config app/.flake8
```

Uma breve explicação sobre cada parte que nem de longe substitui uma necessária lida na documentação que eu aconselho fortemente:

- Na seção `on` definimos os eventos que disparam o fluxo de trabalho, neste caso, em caso de `push` e `pull request` na branch `main`.
- Na seção `strategy` podemos definir uma matriz para diferentes configurações dos jobs. Aqui podemos por exemplo definir diferentes sistemas operacionais, arquiteturas e versões do python para usar no setup, mas optei por usar apenas uma (porquê? senti que deveria poupar recursos do github rs, afinal nesse primeiro momento queria primeiro fazer funcionar). Definindo dois sistemas operacionais diferentes e 3 versões do python, por exemplo, teríamos uma matriz com 6 trabalhos. Como estou usando apenas uma versão do python, na prática estou criando uma matriz de apenas 1 trabalho e não precisaria usar o strategy, mas preferi deixar a estrutura pronta para caso sinta a necessidade.
- As duas primeiras etapas do build são basicamente ações prontas (`actions/checkout@v2` e `actions/setup-python@v2`) que usamos no nosso job, assim apenas referenciamos e não precisamos nos preocupar com algumas etapas usuais do processo.

Abaixo o resultado do job, executando etapa de teste unitário e linting com flake8.

![Logs de um CI simples apenas com a máquina runner](github-actions-first-django-ci.png){: w="500" }

Depois de atualizar o banco de dados do projeto para o PostgreSQL, precisei pesquisar como fazer isso no github actions.

## Adicionando um container de serviço

Com o SQLite, apenas a própria máquina principal com o projeto django é suficiente para rodar os testes. Adicionando um banco como PostgreSQL ou MySQL, precisamos criar um container de serviço, semelhante ao que é feito com o docker-compose. 

### Variáveis de ambiente

Vamos usa um container e a máquina runner (a do projeto django) precisa saber como acessar o banco de dados no container; vamos fazer isso usando variáveis de ambiente. É possível definir variáveis de ambiente no contexto `env` que serão utilizados nos jobs em três níveis do fluxo (com base na própria [documentação](https://docs.github.com/en/actions/learn-github-actions/environment-variables){:target="_blank"}):

- Todo fluxo de trabalho, usando `env` no nível superior do arquivo.
- O conteúdo de um job em um fluxo de trabalho, usando `jobs.<job_id>.env`;
- Uma etapa específica dentro de um job, usando `jobs.<job_id>.steps[*].env`.

Isso é muito importante e ajuda especialmente nessas integrações. Então, defini variáveis de configuração do banco de dados no segundo nível, no escopo do único job do fluxo - elas portanto ficam disponíveis em todo o job, incluindo nas etapas finais em que são executadas as migrações e os testes. Como o nome das variáveis de ambiente utilizadas pela imagem do postgres são diferentes, referenciei com `${{ env.<VAR> }}`. Isso já resolve um eventual problema de credenciais inválidas.

```yaml
name: Django CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:

    runs-on: ubuntu-latest
    env:
      DB_HOST: localhost
      DB_NAME: postgres
      DB_USER: postgres
      DB_PASSWORD: supersecretpassword

    services:
      postgres:
        image: postgres:10-alpine
        env:
          POSTGRES_DB: ${{ env.DB_NAME }}
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
        # Set health checks to wait until postgres has started
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.7]

    steps:
    - uses: actions/checkout@v2
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install postgres prerequisites
      run: sudo apt-get install libpq-dev
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run migrations
      run: python app/manage.py wait_for_db && python app/manage.py migrate
    - name: Run Tests
      run: |
        flake8 --config app/.flake8 && python app/manage.py test app/
```

Há muito mais sobre variáveis de ambiente no github actions que você pode querer saber e a [documentação](https://docs.github.com/en/actions/learn-github-actions/environment-variables){:target="_blank"} é excelente.

Abaixo uma explicação mais pormenorizada sobre o container de serviço e as modificações na máquina runner.

### Container de serviço

Citando a [documentação](https://docs.github.com/en/actions/using-containerized-services/about-service-containers):

> Os containers de serviço são containers do Docker que fornecem uma maneira simples e portátil de hospedar serviços que você pode precisar para testar ou operar seu aplicativo em um fluxo de trabalho. Por exemplo, seu fluxo de trabalho pode precisar executar testes de integração que exijam acesso a um banco de dados e cache de memória.
> 

No meu caso, tinha uma máquina runner com o projeto django e precisava de apenas um container de serviço para o PostgreSQL. A documentação é vasta e inclui um [excelente guia](https://docs.github.com/en/actions/using-containerized-services/creating-postgresql-service-containers){:target="_blank"} para nos ajudar no processo. Minha imagem ficou configurada assim:

```yaml
# jobs.build
    services:
      postgres:
        image: postgres:10-alpine
        env:
          POSTGRES_DB: ${{ env.DB_NAME }}
          POSTGRES_USER: ${{ env.DB_USER }}
          POSTGRES_PASSWORD: ${{ env.DB_PASSWORD }}
        # Set health checks to wait until postgres has started
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
```

As variáveis de ambiente definidas para o job foram acessadas e seus valores foram usados para definição das variáveis de ambiente esperadas pela imagem `postgres:10-alpine`, conforme já explicado. Em `options` podemos usar um recurso para aguardar até o postgres estar pronto. Precisamos também mapear a porta do container de serviço para o host porque, por padrão, o container de serviço não expõe a porta - nada diferente do esperado.

## Requisitos para a máquina runner

Um último e importante ajuste é o de garantir que a máquina runner tenha o que é preciso para usar o PostgreSQL. Precisei instalar as dependências necessárias para o pacote `psycopg2`. Além disso, no CI anterior não tinha a etapa que executava as migrações, então adicionei. Tem-se então as etapas `Install postgres prerequisites` e `Run migrations`.

```yaml
# em jobs.build
  steps:
    - uses: actions/checkout@v2
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install postgres prerequisites
      run: sudo apt-get install libpq-dev
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run migrations
      run: python app/manage.py wait_for_db && python app/manage.py migrate
    - name: Run Tests
      run: |
        flake8 --config app/.flake8 && python app/manage.py test app/
```

O resultado (após, claro, um frutífero processo de falha → mais uma lida na documentação → ajuste → nova tentativa).

![Logs de um CI com o PostgreSQL como container de serviço](github-actions-django-ci-with-postgres.png){: w="500" }

É isso.
{% endraw %}