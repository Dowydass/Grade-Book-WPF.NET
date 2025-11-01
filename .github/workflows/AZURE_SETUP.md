# Azure Deployment Setup Guide

This guide explains how to configure Azure authentication for the GitHub Actions workflow to fix the error:
```
Error: Login failed with Error: Using auth-type: SERVICE_PRINCIPAL. Not all values are present. 
Ensure 'client-id' and 'tenant-id' are supplied.
```

## Prerequisites

1. An Azure subscription
2. Azure CLI installed locally (or use Azure Cloud Shell)
3. Repository admin access to configure secrets

## Step 1: Create an Azure Service Principal

Run the following Azure CLI command to create a service principal:

```bash
az ad sp create-for-rbac --name "grade-book-wpf-deployer" --role contributor --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group-name} --sdk-auth
```

Replace `{subscription-id}` and `{resource-group-name}` with your actual values.

This command will output JSON with the credentials you need:

```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  ...
}
```

## Step 2: Configure GitHub Secrets

Go to your GitHub repository settings and add the following secrets:

1. **AZURE_CLIENT_ID**: Copy the `clientId` value from the JSON output
2. **AZURE_TENANT_ID**: Copy the `tenantId` value from the JSON output
3. **AZURE_SUBSCRIPTION_ID**: Copy the `subscriptionId` value from the JSON output

### How to add secrets:

1. Navigate to your repository on GitHub
2. Go to **Settings** → **Secrets and variables** → **Actions**
3. Click **New repository secret**
4. Add each secret with the name and value as specified above

## Step 3: Verify the Configuration

The workflow file `.github/workflows/azure-deploy.yml` is now configured to use these secrets:

```yaml
- name: Azure Login
  uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

## Alternative: Using Federated Credentials (Recommended)

For enhanced security, you can use OpenID Connect (OIDC) instead of client secrets:

```bash
az ad sp create-for-rbac --name "grade-book-wpf-deployer" \
  --role contributor \
  --scopes /subscriptions/{subscription-id}/resourceGroups/{resource-group-name}
```

Then configure federated credentials:

```bash
az ad app federated-credential create \
  --id {app-id} \
  --parameters '{
    "name": "github-federated-credential",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:{org}/{repo}:ref:refs/heads/main",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

## Troubleshooting

### Error: "Not all values are present"

This error occurs when one or more of the required secrets are missing or not configured correctly. Ensure:
- All three secrets (AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID) are set
- Secret names match exactly (case-sensitive)
- Secret values don't contain extra spaces or characters

### Error: "Authentication failed"

- Verify the service principal has appropriate permissions
- Check that the subscription ID is correct
- Ensure the service principal hasn't expired

## Additional Resources

- [Azure Login Action Documentation](https://github.com/Azure/login#readme)
- [Azure Service Principal Documentation](https://docs.microsoft.com/en-us/azure/active-directory/develop/app-objects-and-service-principals)
- [GitHub Actions Secrets Documentation](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
