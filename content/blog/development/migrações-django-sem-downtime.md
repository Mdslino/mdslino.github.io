+++ 
draft = false
date = 2023-01-23T12:16:17-03:00
title = "Migrações no Django sem Downtime"
description = "Como realizar migrações no Django sem causar indisponibilidade no sistema"
slug = "migrações-django-sem-downtime"
authors = ["Marcelo Lino"]
tags = ["django", "migrations", "downtime", "python"]
categories = ["python", "django", "programação", "devops"]
externalLink = ""
series = ["Docker"]
+++

Uma das coisas mais ferramentas mais poderosas do framework _Django_ é o seu _ORM_ e o gerenciamento de migrações. Porém quando enfrentamos um ambiente produtivo, por diversas vezes temos que executar migrações que adicionam, removem ou alteram colunas, o que causa uma indisponibilidade no sistema, até que o novo código que comporte essa nova estrutura do modelo esteja disponível para atender as requisições.

O fluxo de implantação de um novo código de aplicação Django geralmente consiste em:

{{<mermaid>}}
flowchart TB

Old_Django_Code(Execução Código Atual)
Migration_Execution[(Execução da Migração)]
New_Django_Code(Execução Novo Código)

Old_Django_Code --> Migration_Execution --> New_Django_Code
{{< /mermaid >}}

O problema acontece pois entre a ___Execução da Migração___ e a ___Execução do Novo Código___ o ___Código Atual___ ainda estará em execução até que o novo possa assumir, fazendo com que o _Django_ dispare um `IntegrityError` deixando todas as suas gravações de dados indisponíveis nesse período de tempo.

Pra solucionar esse problema temos alguns procedimentos que podemos seguir.

## Adicionando Campos ou Tabelas

### Adicionar um campo `NULL` ou nova tabela

1. Crie o modelo ou a coluna no seu código;
2. Gere as respectivas migrações com `python manage.py makemigrations`;
3. Faça duas _Pull Requests_, uma contendo a alteração do modelo e outra com a respectiva migração;
4. Faça o _Merge_ e a implantação da _Pull Request_ com a migração;
5. Execute a migração no seu banco de dados de produção. Adição de campos não deve causar problemas se o modelo em produção não esperar que esses campos estejam lá;
6. Faça o _Merge_ da _Pull Request_ com a mudança no modelo;
7. Implante o código com a mudança do modelo. A partir de agora o código vai começar a ler e escrever na nova coluna.

### Adicionando um campo `NOT NULL`

Para iniciar a adição de um campo __NOT NULL__, primeiro siga os passos acima adicionando ele como __NULL__ inicialmente

1. Altere a coluna no modelo para `null=False`. Certifique-se que no seu código você não está inserindo ou atualizando esse campo com valores nulos;
2. Gere as respectivas migrações com `python manage.py makemigrations`;
3. Faça duas _Pull Requests_, uma contendo a alteração do modelo e outra com a respectiva migração;
4. Faça o _Merge_ da _Pull Request_ com a mudança no modelo;
5. Implante o código com a mudança do modelo. A partir de agora o código vai começar a gravar na nova coluna.
6. Faça o _Merge_ e a implantação da _Pull Request_ com a migração;
7. Execute a migração no seu banco de dados de produção.

#### Adicionando `NOT NULL` em tabelas com muitos dados

As migrações do Django insistem que se exista um valor default para colunas com a _constraint_ `NOT NULL`. Esse valor será usado para popular as colunas vazias durante a migração. Esse processo de atualização efetuará um `LOCK` na tabela para escrita até que a atualização esteja concluída. Para tabelas pequenas com menos de 100.000 linhas, provavelmente não haverá problemas, porém para colunas com mais dados isso pode causar uma indisponibilidade do sistema até que a tabela esteja novamente disponível.

Afim de evitar esse comportamento, podemos executar [__Migrações Não Atômicas__](https://docs.djangoproject.com/pt-br/4.1/howto/writing-migrations/#non-atomic-migrations), simplesmente adicionando o atributo `atomic = False` a sua migração, como no exemplo a seguir.

```python
from django.db import migrations

class Migration(migrations.Migration):
    atomic = False
```

## Removendo Campos ou Tabelas

### Removendo um campo `NULL` ou uma Tabela

Basicamente é o mesmo procedimento de adição, porém com os passo em uma ordem um pouco diferente.

1. Remova todas as utilizações do modelo ou coluna a serem removidas da sua aplicação;
2. Faça a implantação de uma versão que ainda tenha a definição da coluna ou modelo a ser removida, porém que não tenha mais a sua utilização;
3. Faça a remoção da coluna ou modelo do seu código;
4. Gere a migração para a remoção com `python manage.py makemigrations`;
5. Faça o _Merge_ da mudança do modelo;
6. Implante o código da mudança do modelo. Certifique-se que não está ocorrendo nenhum erro por ainda existir alguma referência ao código que não mais existe;
7. Faça o _Merge_ da migração;
8. Implante o código da migração;
9. Execute a migração no seu banco de dados.

### Removendo um campo `NOT NULL`

Para remover um campo não nulo, primeiramente você tem que transformá-lo em um campo nulo, então seguir os passos a cima para remover um campo não nulo. Os passos para transformar o campo não nulo em nulo são:

1. Alterar o seu campo para `null=True`;
2. Criar as migrações para esse mudança;
3. Criar duas _Pull Requests_ separadas, uma para a mudança no modelo e outra para migração;
4. Fazer o _Merge_ e implantar o _Pull Request_ com a migração;
5. Execute a migração no banco de dados;
6. Faça o _Merge_ e a implantação da mudança do modelo;
7. [Execute os passos para remover um campo nulo](#removendo-um-campo-null-ou-uma-tabela)
