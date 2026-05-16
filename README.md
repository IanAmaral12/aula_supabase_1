# Trabalhando Com Diferentes Ambientes No Supabase Cloud

![An git flow image from supabase doc](/assets/images/enviroments-git-flow.png "An git flow image from supabase doc")

Esse é um guia detalhado de como trabalhar com ambiente local do supabase e depois fazer o deploy em produção, sem ambiente de homologação. Estou usando como base a [documentação oficial do Supabase](https://supabase.com/docs)

## Criando Projeto do Supabase e Repo Remoto
O primeiro passo é acessar o supabase cloud e criar um novo projeto 
<table>
  <th><img width="842" height="573" alt="image" src="https://github.com/user-attachments/assets/41628107-73c4-4954-824f-392d53b4d0b4" />
</th>
  <th><img width="919" height="783" alt="image" src="https://github.com/user-attachments/assets/1411cd25-9a10-484c-b847-dd282583ab92" />
</th>
</table>

Depois crie uma pasta no seu computador para conter o projeto, então abra o git bash nessa pasta e execute (tenha o node.js instalado):

```bash
git init # cria repo local do git
npm install supabase --save-dev # instala supabase via npm e coloca como dependencia do node
```

Então crie o repo remoto no github, se conecte e faça o push do commit inicial em main (pode adicionar um .gitignore com o node_modules/)

```bash
git branch -M main
git remote add origin <URL_DO_REPO>
git add .
git commit -m'feat:initial dependencies'
git push -u origin main
```

## Conectando Supabase Cloud com Repo Remoto do Supabase

Comece executando o seguinte comando, a pasta /supabase irá aparecer
```bash
npx supabase init
```

Vá até o dashboard do projeto no supabase cloud > settings > integrations:

<img width="829" height="113" alt="image" src="https://github.com/user-attachments/assets/be94c3af-2774-41ba-b202-dc1026d56ee6" />

Então faça o login com o github e de permissão ao supabase para acessar o repo que foi criado, depois configure dessa maneira:

<img width="843" height="652" alt="image" src="https://github.com/user-attachments/assets/b0cf2aee-6a8a-4fee-a551-f69caacb9548" />

> *Importante* lembre de deixar o Working directory como **supabase/**

## Baixando alterações do supabase cloud para o repo local e sincronizando com o repo remoto

Agora a main está definida como branch referente ao ambiente de produção do supabase cloud, mas eles ainda não estão sincronizados e o deploy ainda não está automatizado, vamos trabalhar primeiro na sincronização.

No seu projeto abra o terminal e execute:
```bash
npx supabase login # pressione enter e faça login com o browser
```

Depois é necessário linkar o projeto do supabase

```bash
npx supabase link --project-ref $PROJECT_ID # O project id é o caminho após project/ na url da versão cloud do supabase
```

Agora vamos baixar as alterações do cloud para o repo local
```bash
npx supabase pull
```

Com as alterações locais feitas, vamos atualizar nosso main do repo remoto:

```bash
git add .
git commit -m'feat: supabase cloud initial pull'
git push
```

> Isso fará com que o github e o supabase cloud estejam com a mesma versão sincronizada na branch main

## Configurando CI para automação do deploy na main

Agora precisamos automatizar o processo de deploy no supabase cloud, para isso vamos criar um diretório *.github/workflows* na raiz do projeto, crie dois arquivos:

### ci.yaml
```bash
name: CI

on:
  pull_request:
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: supabase/setup-cli@v1
        with:
          version: latest

      - name: Start Supabase local development setup
        run: supabase db start

      - name: Verify generated types are checked in
        run: |
          supabase gen types typescript --local > types.gen.ts
          if ! git diff --ignore-space-at-eol --exit-code --quiet types.gen.ts; then
            echo "Detected uncommitted changes after build. See status below:"
            git diff
            exit 1
          fi
```

### production.yaml
```bash
name: Deploy Migrations to Production

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    env:
      SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
      SUPABASE_DB_PASSWORD: ${{ secrets.PRODUCTION_DB_PASSWORD }}
      SUPABASE_PROJECT_ID: ${{ secrets.PRODUCTION_PROJECT_ID }}

    steps:
      - uses: actions/checkout@v4

      - uses: supabase/setup-cli@v1
        with:
          version: latest

      - run: supabase link --project-ref $SUPABASE_PROJECT_ID
      - run: supabase db push
```
> Esses arquivos acionam o github actions toda vez que houver uma mudança na branch main do repo remoto e faz o deploy

O production.yaml exige 3 secrets que podem ser configurados diretamente no github em https://github.com/{{seu_usuario}}/{{seu_repo}}/settings/secrets/actions

<img width="786" height="270" alt="image" src="https://github.com/user-attachments/assets/d375217a-be75-4f69-b2d1-f67d7845ca0d" />

> O SUPABASE_ACCESS_TOKEN pode ser gerado em: https://supabase.com/dashboard/account/tokens

Depois configure, faça o push:
```bash
git add .
git commit -m'feat: CI Supabase deplot'
git push
git pull # O pull vai identificar que as variaveis de ambiente foram configuradas no github, se não o vscode vai ficar alertando
```

## Trabalhando com ambiente Local / Produção

Tudo o que foi feito até então, é configuração para o que de fato importa que é o processo de desenvolver localmente, depois fazer o deploy em produção.

Então para começar, vamos fazer o start do serviços locais (tenha o docker instalado e se for windows garanta que o docker desktop está aberto):
```bash
npx supabase init
```

Depois de iniciar os serviços, acesse o *Studio* local (Por padrão: http://127.0.0.1:54323)

Ao acessar o supabase, crie tabelas, colunas, as alterações que desejar no schema *public*:

<img width="1791" height="762" alt="image" src="https://github.com/user-attachments/assets/b0cc0c04-6e50-434b-9eec-b954d19daa2e" />

Agora com o supabase local alterado, temos que criar o arquivo de migrações, para fazer deploy em produção. Faça isso na branch *develop*:

```bash
git branch develop
git checkout develop
npx supabase db diff --use-migra -f first_migrations # -f é o nome que usará para identificar o arquivo de migração
```

Depois, faça o push na propria develop

```bash
git add .
git commit -m'feat: first migration'
git push -u origin develop
```

Agora com as alterações feitas na develop, podemos abrir um *pull request* ou simplesmente fazer um *merge* na main (se tiver permissão pra isso):

```bash
git checkout main
git merge develop
git push -u origin main
```

> Agora você fez alterações locais e subiu pra main. Mas é necessário confirmar se o deploy foi feito.

<img width="1178" height="571" alt="image" src="https://github.com/user-attachments/assets/29774e8a-0016-46a9-beec-22c204a01479" />

Se passar nos *checks* é porque foi para produção.







