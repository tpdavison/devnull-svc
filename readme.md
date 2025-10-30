# Dev Null Service

This service exposes endpoints that accept integration events and discards them after logging. It's useful for local testing, demos, or as a placeholder sink.

## Tech stack

- Azure Functions v4 (Node.js)
- TypeScript (compiled to `dist/`)
- Azure Functions Programming Model v4 (`@azure/functions`)

## Local setup

1. Install dependencies

```bash
npm install
```

2. Create local settings from template

```bash
cp local.settings.template.json local.settings.json
```

3. Build

```bash
npm run build
```

4. Start the Functions host

```bash
npm start
```

Host listens on `http://localhost:7071` by default.

## Deploy to Azure

1. Sign in and select subscription

```bash
az login
az account show
```

2. Create resource group, storage, and Function App (Consumption plan example)

```bash
az group create \
  --name <your-resource-group> \
  --location <permitted-location>
```

```bash
az storage account create \
  --name <yourfuncstorageaccount> \
  --location <permitted-location> \
  --resource-group <your-resource-group> \
  --sku Standard_LRS
```

```bash
az functionapp create \
  --name <your-function-app> \
  --resource-group <your-resource-group> \
  --storage-account <yourfuncstorageaccount> \
  --consumption-plan-location <permitted-location> \
  --runtime node \
  --functions-version 4
```

3. Publish

```bash
func azure functionapp publish <appname>
```

4. Read the host auth key

```bash
az functionapp keys list \
  --name <your-function-app> \
  --resource-group <your-resource-group> \
  --query masterKey \
  --output tsv
```

## Endpoints

- Route prefix is disabled in `host.json`, so there is no `/api` prefix.

### Product Updated

- Method: `POST`
- Path: `/integration/events/product-updated`
- Auth: function-level (host or function keys). Locally you can call without a key if the host is configured to allow it.
- Body schema (JSON):

```json
{
  "id": "string",
  "name": "string",
  "pricePence": 1234,
  "description": "string",
  "updatedAt": "2025-01-01T12:34:56.000Z"
}
```

Notes:

- `updatedAt` must be an ISO 8601 string (Date.toISOString format).
- The service validates the payload and returns 400 on validation errors; otherwise 202 Accepted.

Sample payload file: `samples/post-product-updated.json`.

Without key locally:

```bash
curl -i -X POST http://localhost:7071/integration/events/product-updated \
  -H "Content-Type: application/json" \
  --data-binary @samples/post-product-updated.json
```

With host key in Azure:

```bash
curl -i -X POST "https://<appname>.azurewebsites.net/integration/events/product-updated" \
  -H "Content-Type: application/json" \
  -H "x-functions-key: <hostkey>" \
  --data-binary @samples/post-product-updated.json
```

Expected response:

```http
HTTP/1.1 202 Accepted
{"message":"accepted"}
```

Validation error example:

```http
HTTP/1.1 400 Bad Request
{"error":"ValidationError","details":["updatedAt must be an ISO 8601 string (Date.toISOString)"]}
```
