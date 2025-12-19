## Permission issue 
https://learn.microsoft.com/en-us/azure/cosmos-db/nosql/how-to-connect-role-based-access-control?pivots=azure-cli#permission-model
- Role definitions
az cosmosdb sql role definition list --account-name "cosmoslab82658" --resource-group "cosmos-ws"
az cosmosdb sql role assignment list --account-name "cosmoslab82658" --resource-group "cosmos-ws"
[]  ==> empty means no assignment 

- Role assignments 
az cosmosdb sql role assignment list --account-name "cosmoslab82658" --resource-group "cosmos-ws"

PRINCIPAL_ID=$(az ad signed-in-user show --query id -o tsv)

az cosmosdb sql role assignment create \
  --account-name "cosmoslab82658" \
  --resource-group "cosmos-ws" \
  --role-definition-id "00000000-0000-0000-0000-000000000002" \
  --scope "/subscriptions/[REPLACE WITH YOUR SUBSCRIPTION_ID]/resourceGroups/cosmos-ws/providers/Microsoft.DocumentDB/databaseAccounts/cosmoslab82658" \
  --principal-id "$PRINCIPAL_ID"
