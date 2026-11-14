# AWS Batch Data Engineering Infrastructure

This project provisions the AWS resources needed for the `batch_data_engineering_aws` workload and deploys them through GitHub Actions.

It creates the networking, storage, and compute layers for each environment and keeps deployments controlled by branch and environment naming.

## What it does

This repo sets up the AWS foundation for the data engineering workload:
- GitHub OIDC and IAM setup at the account level
- VPC, subnets, routing, and security groups 
- S3 buckets/Glue catalogs and Secrets Manager entries
- Glue jobs/Step Functions/ SNS topics and Redshift resources
- GitHub Actions pipeline to deploy everything in order

This repo is meant to be used alongside the application code repo that runs the actual batch jobs.

## Resource map

The diagram below shows the main AWS resources this project creates and marks whether each one is account-level/shared or environment-specific.

```mermaid
flowchart LR
    subgraph ACCOUNT[Account-level / shared resources]
        GH[GitHub repo<br/>dev / prod / future env branches]
        GHA[GitHub Actions pipeline]
        OIDC[GitHub OIDC provider<br/>account-level]
        IAM[GitHub deploy IAM role<br/>account-level]
        RG[Default Redshift IAM role<br/>account-level]
        SETUP[setup.yml<br/>account bootstrap]
    end

    subgraph ENV[Environment-specific resources<br/>one per env: dev / test / uat / prod]
        VPC[VPC<br/>shared or env-scoped]
        SUB[Subnets<br/>multi-env / cost-optimized]
        SG[Security groups]
        RT[Route tables / network access]
        S3[S3 buckets<br/>landing / processed / scripts / artifacts]
        PAT[Secrets Manager<br/>github-pat-<env>]
        GLUE[Glue jobs / crawlers]
        SFN[Step Functions state machines]
        SNS[SNS topics]
        REDSHIFT[Redshift namespace]
    end

    GH --> GHA
    GHA --> OIDC
    GHA --> SETUP
    GHA --> NET[networking.yml]
    GHA --> STG[storage.yml]
    GHA --> COMP[compute.yml]

    OIDC --> IAM
    SETUP --> IAM
    SETUP --> RG

    NET --> VPC
    NET --> SUB
    NET --> SG
    NET --> RT

    STG --> S3
    STG --> PAT
    STG --> IAM

    COMP --> GLUE
    COMP --> SFN
    COMP --> SNS
    COMP --> REDSHIFT
    COMP --> IAM
    REDSHIFT --> RG

    classDef account fill:#eaf2ff,stroke:#1f3a5f,stroke-width:1px,color:#10233d;
    classDef env fill:#eafaf1,stroke:#1b7a4a,stroke-width:1px,color:#123d2b;
    classDef stack fill:#fff6dc,stroke:#9b6f00,stroke-width:1px,color:#4c3400;

    class GH,GHA,OIDC,IAM,RG,SETUP account;
    class VPC,SUB,SG,RT,S3,PAT,GLUE,SFN,SNS,REDSHIFT env;
    class NET,STG,COMP stack;
```

### Scope by resource type

- Account-level / shared: GitHub repo, GitHub Actions pipeline, OIDC trust, IAM roles, default Redshift IAM role
- Environment-specific: S3 buckets, Secrets Manager entries, Glue jobs, Step Functions, SNS topics, Redshift namespaces
- Mixed: networking resources can be shared or environment-scoped depending on cost and design, but the repo treats subnets as a multi-environment optimization


## Assumptions

This setup assumes:
- one AWS account for all environments
- one AWS region for all environments
- `dev` is the first environment deployed
- higher environments like `test`, `uat`, and `prod` follow the same pattern later

The repo only deploys from environment branches such as:
- `dev`
- `test`
- `uat`
- `prod`


Feature branches and bugfix branches are not used for deployment

## Naming conventions

1. Stack names use the pattern `-<branchName>`, except `setup.yml`, which is account-level.
2. Environment-specific resources like S3 buckets and Glue jobs use the suffix `-<env>`.
3. Shared resources are kept reusable where possible to reduce cost.

## How to use it

### 1. Set up GitHub OIDC in AWS

Log in to the target AWS account with a privileged admin user and configure GitHub OIDC authentication.

Then run `setup.yml` manually in CloudFormation and set the initial environment parameter to `dev`.

This creates the IAM role GitHub Actions uses to deploy to AWS. That role:
- is mapped to the GitHub identity provider
- is used for AWS deployments (code and infra) from GitHub Actions. Infra deployed from this repo, while source code is deployed from `batch_data_engineering_aws`
- is created once per AWS account, not per environment
- is reused across environments in the same account

This role also becomes the default IAM role for the Redshift namespaces.

References:
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html#manage-oidc-provider-console
- https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws#adding-the-identity-provider-to-aws

Important:
- `setup.yml` is only run for the first environment (`dev`)
- later environments reuse the same account-level role

### 2. Store the GitHub PAT in AWS Secrets Manager

This project needs a GitHub PAT to call the GitHub REST API during ingestion jobs. The secret should not be stored in the repo.

To reuse this flow:
- create the secret in GitHub under `Settings -> Secrets and variables -> Actions -> Repository secrets`
- the GitHub Actions workflow automatically reads the repository secret, passes it to the deployment process, and the `storage.yml` stack stores it in AWS Secrets Manager

### 3. Trigger the deployment

After the IAM trust and secret are in place:

1. clone the infrastructure repo
2. commit the changes
3. push to the `dev` branch
4. GitHub Actions triggers automatically
5. the pipeline deploys the remaining stacks in this order:
   - networking
   - storage
   - compute

The order matters because some resources depend on earlier stacks being present.


## Deployment flow

The execution flow is:
1. `setup.yml` runs once per AWS account for `dev`
2. the GitHub PAT is stored in AWS Secrets Manager and referenced by GitHub Actions
3. a push to `dev` triggers the pipeline
4. CloudFormation deploys the stacks in order
5. environment-specific and account specific resources are created with the correct naming convention

## Operational notes

- feature branches do not trigger deployment. Currently it can deploy through dev and prod branch.
- stack names and resource names stay consistent with the naming pattern
- secrets are not hardcoded into code or config files
- deployment order is part of the setup

## Summary

This repo creates the AWS foundation for the batch data engineering workload and gives you a repeatable way to deploy it through GitHub Actions. It handles the infrastructure layer so the application code can be deployed and run in a clean environment.




