# Backend Service CI/CD Example

## Project

This project provides Azure Functions for managing products.

### Available Endpoints

- `GET /api/products` - List all products
- `POST /api/products` - Upsert (create or update) a product
- `PUT /api/products` - Upsert (create or update) a product

## CI/CD Features

This project includes a GitHub Actions workflow (`.github/workflows/build-and-deploy.yml`) to build the project. This would act as a Continuous Integration (CI) check to verify the latest code changes (git commit) have not broken the code base and a Continuous Deploymemt (CD) to a cloud-based test environment.

> There may be other branches in this repo demonstrating additional CI/CD features.

## Local Setup

1. Install dependencies:

```bash
npm install
```

2. Create your local settings file:

Copy the template to create your local development settings:

```bash
cp local.settings.template.json local.settings.json
```

> **Note:** Never commit your `local.settings.json` to source control. The template is safe to share.

3. Build the project:

```bash
npm run build
```

## Local testing with curl

### Start the Function App

```bash
npm start
```

The function app will start on `http://localhost:7071`.

### List Products

Split the VS Code terminal so you can see the output from the localling running app whilst having a new shell prompt to make a test HTTP call:

```bash
curl -i http://localhost:7071/api/products
```

The fake repo initialises with some example data, so expect an array of products.

### Upsert a Product

Using the sample data file (and a new bash terminal):

```bash
curl -i -X POST http://localhost:7071/api/products \
  -H "Content-Type: application/json" \
  -d @samples/product-post.json
```

Repeating the list products call should now show the new item.

## Azure Setup

### Sign into Azure CLI

Prepare for using the az CLI commands.

1. Ensure you are signed in:

```bash
az login
az account show
```

You should see your account properties displayed if you are successfully signed in.

2. Ensure you know which locations (e.g. uksouth) you are permitted to use:

```bash
az policy assignment list \
  --query "[?name.contains(@, 'sys.regionrestriction')].parameters.listOfAllowedLocations.value | []" \
  -o tsv
```

### Create a Resource Group and Azure Function App

1. Create a resource group (if you do not already have one for this deployment):

```bash
az group create \
  --name <your-resource-group> \
  --location <permitted-location>
```

Remember to follow our naming convention, e.g. `shopping-test-ab47-rg` since we are creating a test environment deployment to which to auto-deploy.

2. Create a storage account (required for Azure Functions):

```bash
az storage account create \
  --name <yourfuncstorageaccount> \
  --location <permitted-location> \
  --resource-group <your-resource-group> \
  --sku Standard_LRS
```

3. Create the Function App:

```bash
az functionapp create \
  --name <your-function-app> \
  --resource-group <your-resource-group> \
  --storage-account <yourfuncstorageaccount> \
  --consumption-plan-location <permitted-location> \
  --runtime node \
  --functions-version 4
```

### Configure Automated Deployment

1. Get the Publishing Profile for your Function App:

   ```bash
   # enable publishing via a shared secret
   az resource update \
    --resource-group <your-resource-group> \
    --namespace Microsoft.Web \
    --parent sites/<your-function-app> \
    --resource-type basicPublishingCredentialsPolicies \
    --name scm \
    --set properties.allow=true

   # query the publishing profile (shared secret)
   az webapp deployment list-publishing-profiles \
    --resource-group <your-resource-group> \
    --name <your-function-app> \
    --xml
   ```

2. Create a GitHub repo secret with the Publishing Profile:

   From the GitHub repo website: `Settings → Secrets and Variables → Actions → New Repository Secret`

   Name the secret `AZURE_FUNCTIONAPP_PUBLISH_PROFILE` and copy in the contents of the publishing profile from the previous step.

3. Update the workflow with your deployed Function App name:

   Modify `.github/workflows/build-and-deploy.yml` with the name of your Azure Function App resource.

   Merge the changes into the main branch to become permenent.

### Configure CORS (if app needs to call this)

If needed, allow cross-domain calls from your app domain and/or localhost, for example:

```bash
az functionapp cors add \
  --name <your-function-app> \
  --resource-group <your-resource-group> \
  --allowed-origins http://localhost:5173
```
