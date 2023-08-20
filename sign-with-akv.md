# Sign and verify artifact with Notation in GitHub Actions

Notation has the following three GitHub Actions available to use.

- `setup`: Install Notation CLI
- `sign`: Sign an OCI artifact with a specified plugin
- `verify`: Verify a signature

It helps software producer and software consumer to automate the Notation installation, signing and verification process respectively.

## Scenario

- Open-source project maintainers want to sign their released software assets including binaries and container images with their private key, assuring authenticity and integrity
- Open-source project users want to verify if the release assets are produced by the official community and are not tampered with

## Hands-on steps

Users can define a GitHub Actions [workflow](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/introduction-to-github-actions) to use the Notation actions to sign and verify artifact in CI/CD pipeline. There is a sample workflow file including three actions for testing purposes. Generally, `sign` are `verify` are used by different type of users, such as software producer and software consumer.

The sample workflow uses Notation Azure Key Vault plugin as an example plugin to sign and verify the latest build of the project, and push the signed image and its signature to Azure Container Registry using Notation Github Actions.

### Prerequisites

- You have created a Key Vault in Azure Key Vault, and created a self-signed signing key and cert by following this [doc](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-sign-build-push) for testing purposes
- You have created a registry in Azure Container Registry
- You have a GitHub repository to store the sample workflow and GitHub Secret, as well as the public certificate and trust policy for verification use

### Add provider credentials to Github Secrets

Add two credentials to authenticate with ACR and AKV as follows, two Github Secrets are required in the whole process:

- `ACR_PASSWORD`: the password to log in to the ACR where your artifact will be released
- `AZURE_CREDENTIALS`: the credential to AKV where your key pair is stored
    
### Create service principal and generate `AZURE_CREDENTIALS`

- Execute the following command to generate Azure credentials. 

```
# login using your own account
az login

# Create a service principal
spn=notationtest
az ad sp create-for-rbac -n $spn --sdk-auth
```

    > [!IMPORTANT]
    > 1. Add the JSON output of the above `az ad sp` command to [Github Secret](https://learn.microsoft.com/en-us/azure/developer/github/connect-from-azure?tabs=azure-portal%2Cwindows#add-the-service-principal-as-a-github-secret) with name `AZURE_CREDENTIALS`.
    >
    > 2. Save the `clientId` from the JSON output into an environment variable (without double quotes) as it will be needed in the next step:
    >```
    >    clientId=<clientId_from_JSON_output_of_last_step>
    >```

### Create GitHub Secret to store credentials 

- Create an encrypted secret to store the Azure Credentials in your own GitHub repository. See [GitHub Docs](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository) for details.

- Add the JSON output of the following `az ad sp` command to the value of GitHub Secret. Naming the GitHub Secret as `AZURE_CREDENTIALS`. 

- Similarly, create another encrypted secret to store the registry credential or password in your own GitHub repository. For example, we use `ACR_PASSWORD` to store the password of the ACR registry in the GitHub Repository secret.

### Grant AKV permission to the service principal

Grant AKV permission to the service principal that we created in the previous step.

```
    # set policy for your AKV
    akv=<your_akv_name>
    az keyvault set-policy --name $akv --spn $clientId --certificate-permissions get --key-permissions sign --secret-permissions get
    ```

See [az keyvault set-policy](https://learn.microsoft.com/en-us/cli/azure/keyvault?view=azure-cli-latest#az-keyvault-set-policy) for details.

### Prepare the trust policy and public certificate 

To create a [trust policy](https://github.com/notaryproject/specifications/blob/main/specs/trust-store-trust-policy.md) file and place the public certificate to the specified folder in the GitHub runner machine for signature verification purposes, you need to prepare a trust policy and public certificate in your GitHub repository.

- Refer to this [.github folder structure in the sample repository](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/main/.github/) to create two folders for trust policy and trust store in your own GitHub repository.

- Create a `trustpolicy.yaml` file and define the `registryScopes` to specify which registry's  repository/namespace that you want to verify. Upload `trustpolicy.yaml` file to the trust policy folder in the GitHub repository. See [this trust policy](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/main/.github/trustpolicy/trustpolicy.json) as an example.

- Download your public certificate from AKV, then upload it to the trust store folder. You can refer to [the Azure doc](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-sign-build-push#create-a-self-signed-certificate-azure-cli) to learn more about this step.

```
CERT_ID=$(az keyvault certificate show -n $KEY_NAME --vault-name $AKV_NAME --query 'id' -o tsv)
az keyvault certificate download --file $CERT_PATH --id $CERT_ID --encoding PEM
```

### Edit the sample workflow

- Create a `workflows` folder under `.github` and create a `workflow.yml` to run and test the CI/CD pipeline. You can copy the sample [workflow](https://github.com/notation-playground/notation-integration-with-ACR-and-AKV/blob/main/.github/workflows/test-notation-action.yml) to your own `workflow.yml` file. 

- Update the environmental variables based on your environment. 

## Trigger the workflow

The workflow trigger logic has been set to `on: push` [event](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#using-events-to-trigger-workflows) so the workflow will run when a commit push is made to any branch in the workflow's repository.

Notation will sign every build and push the signed image with its signature to the registry, then verify the signature in the workflow. On success, you will see the image pushed to your ACR with a COSE format signature attached. 

Another use case is to trigger the workflow when a new tag is pushed to the Github repo. See [this doc](https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#example-excluding-branches-and-tags) for details. This use case is typical in the software release process.

## Check the GitHub Actions workflow status

See the workflow logs from the GitHub Actions in your own repository.
