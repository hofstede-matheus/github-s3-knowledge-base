# github-s3-knowledge-base

Atualmente, temos agentes integrados a ferramentas de desenvolvimento que possibilitam usar o modo "Ask" para fazer perguntas à codebase e até explorar o histórico de commits para entender o porquê de certas mudanças terem sido implementadas.

Nesse tutorial, vou mostrar como criar o seu próprio agente para fazer perguntas a uma codebase que está em um repositório do GitHub.

## Sincronizando o repositório do GitHub com S3

Atualmente, das **Data Sources** disponíveis para knowledges bases do **AWS Bedrock**, o **S3** é a mais adequada para esse caso de uso, pois integração com o GitHub ainda não está disponível. Dito isso, vamos sincronizar os arquivos do repositório com um bucket do S3 através de uma action do GitHub Actions.

```
name: Sync to S3
on:
  push:
    branches: ["main"]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Sync files to S3
      run: |
        aws s3 sync ./ s3://${{ secrets.AWS_S3_BUCKET }} --exclude ".git/*" --delete
```

Após o próximo push na branch "main", os arquivos vão aparecer no bucket do S3 escolhido.

É importante mencionar que, para simplificar, estou usando chaves de acesso. No entanto, a maneira recomendada para manter a segurança nas GitHub Actions é usar OIDC. Ficaria algo assim:

```
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME
    aws-region: us-east-1
```

Além da action, deixei apenas um README.md com uma breve descrição do projeto:

# github-s3-knowledge-base

```
This is an example project to show how to integrate a Github Repository with a AWS Bedrock Agents and use the repository as a knowledge base via S3 sync.
```

## Criando a knowledge base

> Antes de tudo, faça o login com uma conta não-root no console da AWS. Além de não ser uma prática segura, a criação de uma knowledge base com uma conta root falhará.

Uma knowledge base é um repositório centralizado de informações que os sistemas de IA usam para recuperar dados relevantes e gerar respostas precisas e contextualizadas.

Para criar e conectar ao bucket **S3**, vá para **Amazon Bedrock > Build > Knowledge Bases** e clique em **Create Knowledge Base with vector store (unstructured data)**.

Em Choose data source type, selecione Amazon S3.

Configure o data source escolhendo o bucket S3. O **Parsing strategy Amazon Bedrock default parser** é suficiente para esse caso simples, já que não vamos usar imagens, PDFs e outros tipos de dados.

O Embeddings model fica a gosto do freguês, mas o **Titan Embeddings G1** funcionou bem para mim quando preparava esse tutorial. Em **Vector store**, recomendo criar um novo vector store e usar o **Amazon Open Search Serverless**.

Revise e crie a knowledge base.

Minimize image
Edit image
Delete image

Add a caption (optional)
Depois que a knowledge base estiver pronta (demora alguns minutos), ao selecioná-la na lista, verá que a coluna Last sync time está em branco. Selecione-a e clique em Sync. Caso contrário, o agente não terá o contexto dela.

## Criando o agente e conectando-o à knowledge base

Agora em Amazon **Bedrock > Build > Agents** e clique em **Create Agent**.

Selecione um modelo, o Amazon Nova Micro (confirme se está disponível na sua região) funcionou bem nos meus testes e é relativamente barato. Adicione as instruções para o agente. Coloquei isso:

```
You are an agent familiar with the repository (project) in your knowledge base. You will receive questions and must provide answers.
```

Clique em **Save** para salvar as mudanças em Agent details.

Em **Knowledge Bases**, adicione a knowledge base que você acabou de criar. Opcionalmente, descreva a natureza dos dados em **Knowledge Base instructions for Agent**.

```
This is a repository of a project.
```

Com a knowledge base conectada, salve as as mudanças novamente e clique em Prepare para "compilar" o agente com o fundation model, instruções, knowledge base e etc.

Clicando em Test, um chat abrirá e você poderá mandar mensagens para essse agente. Tente algo si simples, como:

```
Q: What is this project about?

A: The project is an example repository to show how to integrate a Github Repository with AWS Bedrock Agents and use the repository as a knowledge base via S3 sync.[1]
```

```

Q: How is the github action connecting to AWS and how could this be improved?

A: The GitHub action connects to AWS by using AWS credentials stored as secrets in the GitHub repository. Here is how it works:

...

To improve this setup, consider the following suggestions:- Use IAM roles for Amazon EC2 instead of AWS access keys for a more secure approach.- Implement a review process for changes to the AWS credentials and repository secrets.- Consider using a more granular approach for syncing files to S3, such as syncing only specific directories or files instead of the entire repository.
```

O mais interessante é que ele referencia o arquivo no S3 (s3://github-s3-knowledge-base/README.md) como fonte.

## Mas não já temos ferramentas para isso?

A pergunta que não quer calar: Não é mais fácil usar o Cursor ou o Copilot para fazer perguntas a uma codebase?

Sim, é mais prático no momento de desenvolvimento, mas ao usar knowledge bases, você está usando RAG Semântico, que é significativamente mais eficiente em custo, velocidade e precisão do que enviar todo o contexto.

Para codebases menos voláteis e situações onde o agente não precisa editar código, o RAG Semântico é a melhor opção.

## Como ir além?

Com licença poética para este caso de uso extremamente simples, o céu é o limite do que pode ser feito com essas ferramentas. Até mesmo no caso do "Stack Overflow pessoal", ele pode ser utilizado para o onboarding de novos desenvolvedores, wikis e documentações dinâmicas de projetos.
