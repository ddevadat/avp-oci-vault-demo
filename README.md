# Argocd Vault Plugin (AVP) OCI Vault Demo 

## Outline

1. Deploy ArgoCD and OCI Vault

    - Create vault & secret in OCI

    - Setup IAM Policy for AuthN & AuthZ

    - Install argocd-vault-plugin (AVP)


2. Deploy a simple Git-based Argo CD application

## Deploy ArgoCD and OCI Vault

### Create vault & secret in OCI

```sh
## Create OCI Vault
## Replace <compartment_ocid> with your compartment's OCID
oci kms management vault create --compartment-id <compartment_ocid> \
    --display-name "DemoVault" \
    --vault-type DEFAULT


## Create a Master Key in the Vault
## Replace <vault_ocid> with the OCID of the vault created in the first step.
oci kms management key create --compartment-id <compartment_ocid> \
    --display-name "MyMasterKey" \
    --vault-id <vault_ocid> \
    --protection-mode HSM \
    --key-shape '{"algorithm":"AES","length":256}'

## Create a Secret in the Vault
## Replace <vault_ocid> with the OCID of the vault.
## Replace <key_ocid> with the OCID of the master key.
## Replace <base64_encoded_secret_content> with your secret content encoded in Base64.
oci vault secret create-base64 --compartment-id <compartment_ocid> \
    --display-name "my_secret_password" \
    --vault-id <vault_ocid> \
    --key-id <key_ocid> \
    --secret-content-content "<base64_encoded_secret_content>"
```

### Setup IAM Policy for AuthN & AuthZ

```sh
## Create a Dynamic Group
## Replace <compartment_ocid> with your compartment's OCID
oci iam dynamic-group create --name <dynamic-group-name> \
    --description "Dynamic group for all Kubernetes nodes in the compartment" \
    --matching-rule "ALL {instance.compartment.id = '<compartment_ocid>'}"

## Create a Policy to Allow Access to Secrets
## Replace <compartment_ocid> with the OCID of the compartment where you want the policy to reside.
## Replace <policy-name> with a name for your policy.
## Replace <dynamic-group-name> with the name of the dynamic group you created in step 1.
## Replace <compartment-name> with the name of the compartment containing the secrets you want to allow access to.
oci iam policy create --compartment-id <compartment_ocid> \
    --name <policy-name> \
    --description "Allow Kubernetes nodes to inspect secrets" \
    --statements '["Allow dynamic-group <dynamic-group-name> to inspect secret-family in compartment <compartment-name>"]'

```


### Deploy Argo CD
```sh
## The actual_vault_ocid is the OCID of your OCI Vault.
## The actual_compartment_ocid is the OCID of the compartment where the vault is located.
sed -i 's|<<vault_id>>|<actual_vault_ocid>|g' argocd-setup/argocd-repo-server.yaml
sed -i 's|<<vault_compartment_id>>|<actual_compartment_ocid>|g' argocd-setup/argocd-repo-server.yaml

kustomize build argocd-setup| kubectl apply -f -

kubectl -n argocd port-forward svc/argocd-server 8080:80 &>/dev/null &

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode
```


## Deploy a simple Git-based Argo CD application

### Create our Argo app
```sh
kubectl apply -f apps/nginx-app.yaml
kubectl port-forward deployment/my-nginx 8081:8080
```

