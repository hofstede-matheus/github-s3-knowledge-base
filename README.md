# github-s3-knowledge-base

Currently, we have agents integrated into development tools that enable us to use "Ask" mode to query the codebase and even explore commit history to understand why certain changes were implemented.

In this tutorial, I'll show you how to create your own agent to query a codebase that is in a GitHub repository.

## Syncing the GitHub repository with S3

Currently, among the **Data Sources** available for **AWS Bedrock** knowledge bases, **S3** is the most suitable for this use case, as GitHub integration is not yet available. That said, let's synchronize the repository files with an S3 bucket through a GitHub Actions workflow.

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

After the next push to the "main" branch, the files will appear in the chosen S3 bucket.

It's important to mention that, for simplicity, I'm using access keys. However, the recommended way to maintain security in GitHub Actions is to use OIDC. It would look like this:

```
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME
    aws-region: us-east-1
```

In addition to the action, I've included only a README.md with a brief description of the project:

# github-s3-knowledge-base

```
This is an example project to show how to integrate a Github Repository with a AWS Bedrock Agents and use the repository as a knowledge base via S3 sync.
```

## Creating the knowledge base

> First of all, log in with a non-root account in the AWS console. In addition to not being a secure practice, creating a knowledge base with a root account will fail.

A knowledge base is a centralized repository of information that AI systems use to retrieve relevant data and generate accurate and contextualized responses.

To create and connect to the **S3** bucket, go to **Amazon Bedrock > Build > Knowledge Bases** and click on **Create Knowledge Base with vector store (unstructured data)**.

In Choose data source type, select Amazon S3.

Configure the data source by choosing the S3 bucket. The **Parsing strategy Amazon Bedrock default parser** is sufficient for this simple case, since we won't be using images, PDFs, and other data types.

The Embeddings model is up to you, but **Titan Embeddings G1** worked well for me when preparing this tutorial. In **Vector store**, I recommend creating a new vector store and using **Amazon Open Search Serverless**.

Review and create the knowledge base.

Minimize image
Edit image
Delete image

Add a caption (optional)
After the knowledge base is ready (takes a few minutes), when you select it from the list, you'll see that the Last sync time column is blank. Select it and click Sync. Otherwise, the agent won't have its context.

## Creating the agent and connecting it to the knowledge base

Now go to Amazon **Bedrock > Build > Agents** and click **Create Agent**.

Select a model; Amazon Nova Micro (confirm if it's available in your region) worked well in my tests and is relatively inexpensive. Add the instructions for the agent. I used this:

```
You are an agent familiar with the repository (project) in your knowledge base. You will receive questions and must provide answers.
```

Click **Save** to save the changes in Agent details.

In **Knowledge Bases**, add the knowledge base you just created. Optionally, describe the nature of the data in **Knowledge Base instructions for Agent**.

```
This is a repository of a project.
```

With the knowledge base connected, save the changes again and click Prepare to "compile" the agent with the foundation model, instructions, knowledge base, etc.

By clicking Test, a chat will open and you can send messages to this agent. Try something simple, like:

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

The most interesting thing is that it references the file in S3 (s3://github-s3-knowledge-base/README.md) as the source.

## But don't we already have tools for this?

The burning question: Isn't it easier to use Cursor or Copilot to query a codebase?

Yes, it's more practical during development, but by using knowledge bases, you're using Semantic RAG, which is significantly more efficient in cost, speed, and accuracy than sending the entire context.

For less volatile codebases and situations where the agent doesn't need to edit code, Semantic RAG is the best option.

## How to go further?

With poetic license for this extremely simple use case, the sky's the limit for what can be done with these tools. Even in the case of a "personal Stack Overflow," it can be used for onboarding new developers, wikis, and dynamic project documentation.
