# AWS Batch Data Engineering Infrastructure

This project provisions the AWS resources needed for the `batch_data_engineering_aws` workload and deploys them through GitHub Actions.

It creates the networking, storage, and compute layers for each environment and keeps deployments based on branch and environment naming.

## What it does

This repo sets up the AWS foundation for the data engineering workload:
- GitHub OIDC and IAM setup at the account level
- VPC, subnets, IAM roles, and security groups 
- S3 buckets/Glue catalogs and Secrets Manager entries
- Glue jobs/Step Functions/ SNS topics and Redshift resources
- GitHub Actions pipeline to deploy everything in order

This repo is meant to be used alongside the application code repo (batch_data_engineering_aws) that runs the actual jobs.

## Resource map

The diagram below shows the AWS resources CloudFormation creates in this repo and marks whether they are account-level/shared or environment-specific.

```mermaid
flowchart LR
    subgraph ACCOUNT[Account-level / shared]
        GHA[GitHub Actions pipeline]
        OIDC[GitHub OIDC provider]
        CICD[IAM role: cicd-iam]
        REDSHIFT_ROLE[Default Redshift IAM role]
        SETUP[setup.yml bootstrap]
    end

    subgraph NETWORK[networking.yml]
        VPC[VPC: MyVPC]
        SUB1[Subnet: MyPrivateSubnet1]
        SUB2[Subnet: MyPrivateSubnet2]
        SUB3[Subnet: MyPrivateSubnet3]
        SG[Security group: redshift-sg-${env}]
        GLUE_IAM[Role: glue-iam-${env}]
        STEP_IAM[Role: stepFunction-iam-${env}]
    end

    subgraph STORAGE[storage.yml]
        RAW[S3 bucket: raw-githubanalytics-${env}]
        BRONZE[S3 bucket: bronze-githubanalytics-${env}]
        SILVER[S3 bucket: silver-githubanalytics-${env}]
        SOURCE[S3 bucket: sourcecode-githubanalytics-${env}]
        META[S3 bucket: metadata-githubanalytics-${env}]
        DB_RAW[Glue database: raw-githubanalytics-${env}]
        DB_BRONZE[Glue database: bronze-githubanalytics-${env}]
        DB_SILVER[Glue database: silver-githubanalytics-${env}]
        DB_META[Glue database: metadata-githubanalytics-${env}]
        PAT[Secrets Manager: github_pat_${env}]
        NS[Redshift namespace: redshift-serving-namespace-${env}]
        WG[Redshift workgroup: redshift-serving-workgroup-${env}]
    end

    subgraph COMPUTE[compute.yml]
        JOB1[Glue job: githubanalytics-extract-${env}]
        JOB2[Glue job: githubanalytics-bronzeingestion-${env}]
        JOB3[Glue job: githubanalytics-silvertransform-${env}]
        SFN[Step Functions: orchestrator-workflow-${env}]
        SCH[EventBridge scheduler: github-analytics-schedule-${env}]
        SNS[SNS topic: github-analytics-${env}]
        SUB_EMAIL[SNS email subscription]
    end

    GHA --> SETUP
    GHA --> OIDC
    GHA --> NET[networking.yml]
    GHA --> STG[storage.yml]
    GHA --> COMP[compute.yml]

    SETUP --> CICD
    OIDC --> CICD
    SETUP --> REDSHIFT_ROLE

    NET --> VPC
    NET --> SUB1
    NET --> SUB2
    NET --> SUB3
    NET --> SG
    NET --> GLUE_IAM
    NET --> STEP_IAM

    STG --> RAW
    STG --> BRONZE
    STG --> SILVER
    STG --> SOURCE
    STG --> META
    STG --> DB_RAW
    STG --> DB_BRONZE
    STG --> DB_SILVER
    STG --> DB_META
    STG --> PAT
    STG --> NS
    STG --> WG
    NS --> REDSHIFT_ROLE
    WG --> SG
    WG --> SUB1
    WG --> SUB2
    WG --> SUB3

    COMP --> JOB1
    COMP --> JOB2
    COMP --> JOB3
    COMP --> SFN
    COMP --> SCH
    COMP --> SNS
    SNS --> SUB_EMAIL
    JOB1 --> GLUE_IAM
    JOB2 --> GLUE_IAM
    JOB3 --> GLUE_IAM
    SFN --> STEP_IAM
    SCH --> STEP_IAM

    classDef account fill:#eaf2ff,stroke:#1f3a5f,stroke-width:1px,color:#10233d;
    classDef network fill:#fff4d6,stroke:#9b6f00,stroke-width:1px,color:#4c3400;
    classDef storage fill:#eafaf1,stroke:#1b7a4a,stroke-width:1px,color:#123d2b;
    classDef compute fill:#f8e8ff,stroke:#6e3da7,stroke-width:1px,color:#38204a;

    class GHA,OIDC,CICD,REDSHIFT_ROLE,SETUP account;
    class VPC,SUB1,SUB2,SUB3,SG,GLUE_IAM,STEP_IAM network;
    class RAW,BRONZE,SILVER,SOURCE,META,DB_RAW,DB_BRONZE,DB_SILVER,DB_META,PAT,NS,WG storage;
    class JOB1,JOB2,JOB3,SFN,SCH,SNS,SUB_EMAIL compute;
```

### Resource scope summary

- Account-level / shared: GitHub OIDC, IAM deployment role, default Redshift IAM role, account bootstrap
- Environment-specific: S3 buckets, Secrets Manager entries, Glue databases, Glue jobs, Step Functions, EventBridge schedule, SNS topic, Redshift namespace, Redshift workgroup
- Mixed / shared where applicable: VPC, subnets, and security groups are created as part of the networking stack and can be reused across environments to save cost


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

This project needs a GitHub PAT to call the GitHub REST API during the ingestion job. The secret should not be stored in the repo. Currently, we assume all env have same secret value, so we create only 1 github UI secret, but in secret manager cloudformation creates different secret names based on env.

To reuse this flow:
- create the secret in GitHub under `Settings -> Secrets and variables -> Actions -> Repository secrets`
- the GitHub Actions workflow automatically reads the repository secret, passes it to the deployment process, and the `storage.yml` stack stores it in AWS Secrets Manager per env.

### 3. Trigger the deployment

After step1 and step2:

1. clone the infrastructure repo
2. commit the changes
3. push to the `dev` branch
4. GitHub Actions triggers automatically
5. the cicd pipeline deploys the remaining stacks in this order:
   - networking
   - storage
   - compute

The order matters because some resources depend on the earlier stacks being present.


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

This repo creates the infra for the batch data engineering workload and gives a way to deploy it through GitHub Actions. It handles the infrastructure layer so the application code can be deployed and run in a clean environment.




