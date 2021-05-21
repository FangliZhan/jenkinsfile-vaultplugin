# Jenkins and Vault pipeline example

This is a pipeline example that using your secrets in a Vault instance it is able to:
* Connects to your GitHub account by `curl`request to list your repos
* Connects to Terraform Cloud to list your current workspaces in an Organization

The goal is about showing the Jenkins Vault Plugin in action with an easy pipeline execution

## Prepare your Environment

You need to have an account in Terraform Cloud (TFC) and a Vault server running. If you don't, you can do the following:
* [Sign up into TFC/TFE](https://app.terraform.io/signup) and follow the process to create your first organization
* For a quick shot just run Vault in development mode with (it will run on your localhost on port 9200):
  ```bash
  vault server -dev -dev-listen-address="127.0.0.1:9200" -dev-root-token-id="root"
  ```
> NOTE: When you are running Vault in development mode everything is in memory, so data is not persisted. In that case you need to run Vault in [server mode](https://learn.hashicorp.com/tutorials/vault/getting-started-deploy)


Once you have TFC account and Vault running:

* Create some tokens in your TFC Org:
  - Create an [organization token]({"data":{"gh_user"="dcanadillas"}}
  - Create a [user token](https://www.terraform.io/docs/cloud/users-teams-organizations/api-tokens.html#user-api-tokens)
* Change the values in the included file `secrets.json` with your values (GitHub token, TFC tokens and TFC org name)
* Create your secrets in Vault (here are the commands, but you can do it in the UI):
  ```bash
  export VAULT_ADDR="127.0.0.1:9200"
  ```
  ```bash
  vault login root
  ```
  ```bash
  vault secrets enable -path=kv -version=2 kv
  ```
  ```bash
  vault write kv/data/cicd @secrets.json
  ```
* Enable AppRole auth method so that Jenkins can be authenticated to Vault
  - Enable approle auth method
```
vault auth enable approle
````
 - create jenkins-pipeline policy for Jenkins to access the secrets 
```
vault policy write jenkins-pipeline -<< EOF
path "kv/data/cicd" {
  capabilities = [ "read", "list" ]
}
path "kv/cicd" {
  capabilities = [ "read", "list" ]
}
EOF
```
 - Create an approle for jenkins pipeline
```
 vault write auth/approle/role/jenkins-pipeline \
> token_policies=jenkins-pipeline \
> token_ttl=1h \
> token_max_ttl=4h
```
 - Get the role id and secret id for the role

```
vault read auth/approle/role/jenkins-pipeline/role-id

vault write -f auth/approle/role/jenkins-pipeline/secret-id

```

* Configure your Jenkins with Vault Plugin installed. You can take a look at [this repo]() using JCasC to configure from scratch. I adds the Jenkinsfile pipeline from this repo.
 - Download and install Vault Plugin in Jenkins (Manage Jenkins--> Manage Plugin --> search for the Vault plugin --> install)
 - Create credentials for Vault in Jenkins
    - Navigate to Manage Credentials--> Global Credentials--> Add credentials
    - Select Vault AppRole as the credential type
![image](https://user-images.githubusercontent.com/31291225/119159032-7766ab80-ba1c-11eb-9bfd-4fea3f4908c6.png)
 - Create a pipeline using the Jenkinsfile pipeline code
 - Run the pipeline. The output will look similiar to below when the build is successful
![image](https://user-images.githubusercontent.com/31291225/119159597-01167900-ba1d-11eb-9d29-aab3fdb8c778.png)



