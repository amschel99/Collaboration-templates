# Secret Management with HashiCorp Vault in CI/CD Pipelines

This guide outlines how to use HashiCorp Vault to securely store secrets and automatically inject them into CI/CD pipelines.

## Table of Contents

- [Introduction](#introduction)
- [Setting Up HashiCorp Vault](#setting-up-hashicorp-vault)
- [Storing Secrets in Vault](#storing-secrets-in-vault)
- [Integrating Vault with CI/CD Pipelines](#integrating-vault-with-cicd-pipelines)
  - [GitHub Actions](#github-actions)
  - [GitLab CI](#gitlab-ci)
  - [Jenkins](#jenkins)
  - [CircleCI](#circleci)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Introduction

HashiCorp Vault is a secrets management tool that provides a secure, centralized storage system for sensitive data like API keys, passwords, and certificates. By integrating Vault with your CI/CD pipelines, you can:

- Centralize secret management
- Implement fine-grained access control
- Rotate credentials automatically
- Audit secret access
- Eliminate hardcoded secrets in your codebase and CI/CD configurations

## Setting Up HashiCorp Vault

### Installation

1. **Install Vault Server**:

   ```bash
   # Download and install Vault
   wget https://releases.hashicorp.com/vault/1.14.0/vault_1.14.0_linux_amd64.zip
   unzip vault_1.14.0_linux_amd64.zip
   sudo mv vault /usr/local/bin/
   ```

2. **Configure Vault Server**:
   Create a configuration file `config.hcl`:

   ```hcl
   storage "raft" {
     path    = "./vault/data"
     node_id = "node1"
   }

   listener "tcp" {
     address     = "0.0.0.0:8200"
     tls_disable = 1  # Enable TLS in production
   }

   api_addr = "http://127.0.0.1:8200"
   cluster_addr = "https://127.0.0.1:8201"
   ui = true
   ```

3. **Initialize Vault**:

   ```bash
   vault server -config=config.hcl
   export VAULT_ADDR='http://127.0.0.1:8200'
   vault operator init
   ```

   Save the unseal keys and root token securely. You'll need them to unseal Vault and authenticate.

4. **Unseal Vault**:

   ```bash
   vault operator unseal <unseal-key-1>
   vault operator unseal <unseal-key-2>
   vault operator unseal <unseal-key-3>
   ```

5. **Authenticate**:
   ```bash
   vault login <root-token>
   ```

### Setting Up Authentication Methods

For CI/CD integration, you'll need to set up appropriate authentication methods:

1. **AppRole Authentication** (recommended for CI/CD):

   ```bash
   # Enable AppRole auth
   vault auth enable approle

   # Create a role for your CI/CD pipeline
   vault write auth/approle/role/cicd-role \
       secret_id_ttl=24h \
       token_ttl=1h \
       token_policies=cicd-policy

   # Get RoleID
   vault read auth/approle/role/cicd-role/role-id

   # Generate SecretID
   vault write -f auth/approle/role/cicd-role/secret-id
   ```

2. **Create a Policy**:

   ```bash
   # Create a policy file cicd-policy.hcl
   path "secret/data/cicd/*" {
     capabilities = ["read"]
   }

   # Apply the policy
   vault policy write cicd-policy cicd-policy.hcl
   ```

## Storing Secrets in Vault

1. **Enable the KV Secrets Engine**:

   ```bash
   vault secrets enable -version=2 kv
   vault secrets enable -path=secret kv-v2
   ```

2. **Store Secrets**:

   ```bash
   # Store API keys
   vault kv put secret/cicd/api-keys \
       polymarket_api_key=your-api-key \
       private_key=your-private-key

   # Store database credentials
   vault kv put secret/cicd/database \
       username=db-user \
       password=db-password

   # Store environment-specific configurations
   vault kv put secret/cicd/environments/production \
       rpc_url=https://mainnet.infura.io/v3/your-project-id
   ```

3. **Verify Secrets**:
   ```bash
   vault kv get secret/cicd/api-keys
   ```

## Integrating Vault with CI/CD Pipelines

### GitHub Actions

1. **Store Vault Credentials as GitHub Secrets**:

   - VAULT_ADDR
   - VAULT_ROLE_ID
   - VAULT_SECRET_ID

2. **Workflow Example**:

   ```yaml
   name: Deploy with Vault Secrets

   on:
     push:
       branches: [main]

   jobs:
     deploy:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@v3

         - name: Install Vault CLI
           run: |
             wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
             echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
             sudo apt update && sudo apt install vault

         - name: Fetch secrets from Vault
           id: secrets
           run: |
             export VAULT_ADDR=${{ secrets.VAULT_ADDR }}

             # Authenticate to Vault using AppRole
             VAULT_TOKEN=$(vault write -field=token auth/approle/login \
               role_id=${{ secrets.VAULT_ROLE_ID }} \
               secret_id=${{ secrets.VAULT_SECRET_ID }})

             export VAULT_TOKEN

             # Get secrets and set as environment variables
             echo "PRIVATE_KEY=$(vault kv get -field=private_key secret/cicd/api-keys)" >> $GITHUB_ENV
             echo "RPC_URL=$(vault kv get -field=rpc_url secret/cicd/environments/production)" >> $GITHUB_ENV
             echo "DB_PASSWORD=$(vault kv get -field=password secret/cicd/database)" >> $GITHUB_ENV

         - name: Deploy application
           run: |
             # Use secrets from environment variables
             echo "Deploying with RPC URL: $RPC_URL"
             # Your deployment script here
   ```

### GitLab CI

1. **Store Vault Credentials as GitLab CI Variables**:

   - VAULT_ADDR
   - VAULT_ROLE_ID
   - VAULT_SECRET_ID

2. **Pipeline Example**:

   ```yaml
   stages:
     - deploy

   deploy:
     stage: deploy
     image: hashicorp/vault:latest
     script:
       - apk add --no-cache curl jq
       - |
         # Authenticate to Vault using AppRole
         export VAULT_ADDR=$VAULT_ADDR
         VAULT_TOKEN=$(vault write -field=token auth/approle/login \
           role_id=$VAULT_ROLE_ID \
           secret_id=$VAULT_SECRET_ID)
         export VAULT_TOKEN

         # Get secrets
         PRIVATE_KEY=$(vault kv get -field=private_key secret/cicd/api-keys)
         RPC_URL=$(vault kv get -field=rpc_url secret/cicd/environments/production)
         DB_PASSWORD=$(vault kv get -field=password secret/cicd/database)

         # Export secrets as environment variables
         export PRIVATE_KEY
         export RPC_URL
         export DB_PASSWORD

         # Your deployment script here
         echo "Deploying with RPC URL: $RPC_URL"
   ```

### Jenkins

1. **Install the HashiCorp Vault Plugin**:

   - Go to "Manage Jenkins" > "Manage Plugins" > "Available"
   - Search for "HashiCorp Vault" and install

2. **Configure the Vault Plugin**:

   - Go to "Manage Jenkins" > "Configure System"
   - Find the "Vault" section
   - Add Vault server URL and configure authentication

3. **Jenkinsfile Example**:

   ```groovy
   pipeline {
       agent any

       options {
           // This enables the Vault plugin
           withVault([
               configuration: [
                   vaultUrl: 'http://vault:8200',
                   vaultCredentialId: 'vault-approle',
                   engineVersion: 2
               ],
               vaultSecrets: [
                   [path: 'secret/cicd/api-keys', secretValues: [
                       [envVar: 'PRIVATE_KEY', vaultKey: 'private_key'],
                       [envVar: 'API_KEY', vaultKey: 'polymarket_api_key']
                   ]],
                   [path: 'secret/cicd/environments/production', secretValues: [
                       [envVar: 'RPC_URL', vaultKey: 'rpc_url']
                   ]]
               ]
           ])
       }

       stages {
           stage('Deploy') {
               steps {
                   sh '''
                       echo "Deploying with RPC URL: $RPC_URL"
                       # Your deployment script here
                   '''
               }
           }
       }
   }
   ```

### CircleCI

1. **Store Vault Credentials as CircleCI Environment Variables**:

   - VAULT_ADDR
   - VAULT_ROLE_ID
   - VAULT_SECRET_ID

2. **Config Example**:

   ```yaml
   version: 2.1
   jobs:
     deploy:
       docker:
         - image: cimg/base:2023.03
       steps:
         - checkout
         - run:
             name: Install Vault CLI
             command: |
               wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
               echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
               sudo apt update && sudo apt install vault
         - run:
             name: Fetch secrets from Vault
             command: |
               export VAULT_ADDR=$VAULT_ADDR

               # Authenticate to Vault using AppRole
               VAULT_TOKEN=$(vault write -field=token auth/approle/login \
                 role_id=$VAULT_ROLE_ID \
                 secret_id=$VAULT_SECRET_ID)

               export VAULT_TOKEN

               # Get secrets and set as environment variables
               echo "export PRIVATE_KEY=$(vault kv get -field=private_key secret/cicd/api-keys)" >> $BASH_ENV
               echo "export RPC_URL=$(vault kv get -field=rpc_url secret/cicd/environments/production)" >> $BASH_ENV
               echo "export DB_PASSWORD=$(vault kv get -field=password secret/cicd/database)" >> $BASH_ENV
         - run:
             name: Deploy application
             command: |
               echo "Deploying with RPC URL: $RPC_URL"
               # Your deployment script here

   workflows:
     version: 2
     deploy:
       jobs:
         - deploy
   ```

## Best Practices

1. **Least Privilege Access**: Grant CI/CD pipelines access only to the secrets they need.

2. **Short-lived Tokens**: Configure Vault tokens with short TTLs to minimize the risk of token exposure.

3. **Secret Rotation**: Regularly rotate secrets and credentials.

4. **Audit Logging**: Enable and monitor Vault's audit logs to track secret access.

5. **Separate Environments**: Use different paths in Vault for different environments (dev, staging, prod).

6. **Avoid Printing Secrets**: Never print or log secrets in your CI/CD output.

7. **Secure CI/CD Variables**: Protect the Vault credentials (role ID, secret ID) stored in your CI/CD system.

8. **Dynamic Secrets**: When possible, use Vault's dynamic secret generation for databases and cloud providers.

## Troubleshooting

### Common Issues

1. **Authentication Failures**:

   - Verify that your role ID and secret ID are correct
   - Check if the secret ID has expired (they have TTLs)
   - Ensure the Vault server is unsealed and accessible

2. **Permission Denied**:

   - Check the policy attached to your AppRole
   - Verify the path to your secrets is correct
   - Ensure the policy has the necessary capabilities (read, list, etc.)

3. **Vault CLI Not Found**:

   - Ensure Vault CLI is installed in your CI/CD environment
   - Add the installation step if necessary

4. **Secret Not Found**:
   - Verify the path to your secret
   - Check if you're using the correct version of the KV secrets engine (v1 vs v2)
   - For KV v2, ensure you're using the correct path format (secret/data/path for reading)

### Debugging Tips

1. **Enable Verbose Logging**:

   ```bash
   export VAULT_ADDR='http://127.0.0.1:8200'
   VAULT_TOKEN=$(vault write -field=token auth/approle/login \
     role_id=$VAULT_ROLE_ID \
     secret_id=$VAULT_SECRET_ID)
   export VAULT_TOKEN

   # Add -debug flag for verbose output
   vault kv get -debug secret/cicd/api-keys
   ```

2. **Check Token Capabilities**:

   ```bash
   vault token capabilities secret/cicd/api-keys
   ```

3. **Verify Secret Existence Without Reading**:
   ```bash
   vault kv metadata get secret/cicd/api-keys
   ```
