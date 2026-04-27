---
name: add-teams
description: Add Microsoft Teams channel to NanoPilot — Entra app, Azure Bot, manifest
user_invocable: true
---

# /add-teams — Add Teams Channel

## Pre-flight
1. Check Azure CLI installed: `az version`
2. Check logged in: `az account show`
3. If not: `az login`

## Steps

### 1. Create Entra App Registration
```bash
az ad app create \
  --display-name "NanoPilot" \
  --sign-in-audience AzureADandPersonalMicrosoftAccount
```

Save the `appId` from the output.

Tip: Run `gh copilot explain 'az ad app create --sign-in-audience'` to understand the options.

### 2. Create Service Principal
```bash
az ad sp create --id <appId>
```

### 3. Create Client Secret
```bash
az ad app credential reset --id <appId> --years 2
```

Save the `password` from the output.

### 4. Get Tenant ID
```bash
az account show --query tenantId -o tsv
```

### 5. Write Configuration
Add to `.env`:
```
MICROSOFT_APP_ID=<appId>
MICROSOFT_APP_PASSWORD=<password>
MICROSOFT_APP_TENANT_ID=<tenantId>
```

### 6. Create Azure Bot Resource
```bash
az bot create \
  --resource-group <rg-name> \
  --name nanopilot-bot \
  --app-type SingleTenant \
  --appid <appId> \
  --tenant-id <tenantId>
```

For local dev, use a Dev Tunnel:
```bash
devtunnel create --allow-anonymous
devtunnel port create -p 3978
devtunnel host
```

Update bot messaging endpoint:
```bash
az bot update --resource-group <rg-name> --name nanopilot-bot \
  --endpoint "https://<tunnel-url>/api/messages"
```

Tip: Run `gh copilot suggest 'create an Azure Bot resource for Teams'` for help with the az commands.

### 7. Enable Teams Channel
```bash
az bot msteams create --resource-group <rg-name> --name nanopilot-bot
```

### 8. Generate Teams Manifest
```bash
npm run build-manifest -- nanopilot
```

This creates a ZIP file in `agents/nanopilot/teams-app/`.

### 9. Sideload in Teams
1. Open Teams → Apps → Manage your apps → Upload a custom app
2. Select the generated ZIP file
3. Add the bot to a chat or channel

### 10. Test
Send a message to the bot in Teams and verify the response.

## Troubleshooting
- **401 Unauthorized**: Check MICROSOFT_APP_ID and MICROSOFT_APP_PASSWORD match the Entra app
- **"Bot Framework connector" errors**: Ensure the messaging endpoint URL is correct and reachable
- **Dev Tunnel URL changes**: Update the bot endpoint after each tunnel restart, or use a persistent tunnel
- See `/debug` for more troubleshooting steps
