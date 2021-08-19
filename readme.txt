## For ease of use export a few vars for ease of use

$ export APIM_SERVICE_NAME="<SERVICE_NAME>"
$ export AZ_SUBSCRIPTION_ID="<ID>"
$ export AZ_RESOURCE_GROUP="<GROUP_NAME>"

# Create a new APIM service (This can take up to 45mins)

$ az apim create --name $APIM_SERVICE_NAME \
               --subscription $AZ_SUBSCRIPTION_ID \
               --resource-group $AZ_RESOURCE_GROUP \
               --publisher-email "capodacac@gmail.com" \
               --publisher-name "chris chris"

# update the service name
# apim/api.yaml
...
  - url: http://<SERVICE_NAME>.azure-api.net
  - url: https://<SERVICE_NAME>.azure-api.net
...

# Import the OpenAPI definition into APIM Service:

$ az apim api import --path / \
                   --api-id dapr \
                   --subscription $AZ_SUBSCRIPTION_ID \
                   --resource-group $AZ_RESOURCE_GROUP \
                   --service-name $APIM_SERVICE_NAME \
                   --display-name "Demo Dapr Service API" \
                   --protocols http https \
                   --subscription-required true \
                   --specification-path apim/api.yaml \
                   --specification-format OpenApi

# We will need a APIM Token, export it for later.
$ export AZ_API_TOKEN=$(az account get-access-token --resource=https://management.azure.com --query accessToken --output tsv)


# APIM policies can be found in /apim
/apim/policy-all.json
/apim/policy-echo.json
...

# Apply the global Policy
$ curl -i -X PUT \
     -d @apim/policy-all.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/apis/dapr/policies/policy?api-version=2019-12-01"

## NOTE: Methods on any Dapr service users follow this API:
'POST/GET/PUT/DELETE /v1.0/invoke/<appId>/method/<method-name>'

# apply the echo operation API
$ curl -i -X PUT \
     -d @apim/policy-echo.json \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/apis/dapr/operations/echo/policies/policy?api-version=2019-12-01"

## Time to configure the GateWay

# Create the GateWay
$ curl -i -X PUT -d '{"properties": {"description": "Dapr Gateway","locationData": {"name": "Virtual"}}}' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway?api-version=2019-12-01"


# Map the gateway to our API
$ curl -i -X PUT -d '{ "properties": { "provisioningState": "created" } }' \
     -H "Content-Type: application/json" \
     -H "If-Match: *" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway/apis/dapr?api-version=2019-12-01"

# Deploy the DAPR service
# Ensure the annotations are appropriate
# k8s/echo-service.yaml
...
annotations:
     dapr.io/enabled: "true"
     dapr.io/id: "event-subscriber"
     dapr.io/port: "8080"
...

# Deploy!
$ kubectl apply -f k8s/echo-service.yaml

# Sanity check the pods!
$ kubectl get pods -l demo=dapr-apim -w


# Connect the Self-hosted APIM Gateway to the APIM service in k8
# We need a secret to do this.
$ curl -i -X POST -d '{ "keyType": "primary", "expiry": "2020-10-10T00:00:01Z" }' \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/gateways/demo-apim-gateway/generateToken?api-version=2019-12-01"

# Copy the output of "value" attr, from the response on pass it into k8
# this command should look something like "GatewayKey fjla98u09j09j092j09jf0...."
$ kubectl create secret generic demo-apim-gateway-token --type Opaque --from-literal value="GatewayKey paste-the-key-here"

# Now create a ConfigMap containing the APIM service endpoint that we want to configure the self-hosted gateway:
$ kubectl create configmap demo-apim-gateway-env --from-literal \
     "config.service.endpoint=https://${APIM_SERVICE_NAME}.management.azure-api.net/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}?api-version=2019-12-01"


# We are ready to deploy the gateway + sanity check
$ kubectl apply -f k8s/gateway.yaml
$ kubectl get pods -l app=demo-apim-gateway

# To check logs:
$ kubectl logs -l app=demo-apim-gateway -c demo-apim-gateway


# API Test
# Export the IP address to a variable
$ export GATEWAY_IP=$(kubectl get svc demo-apim-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# API Subscription Key
$ curl -i -H POST -d '{}' -H "Authorization: Bearer ${AZ_API_TOKEN}" \
     "https://management.azure.com/subscriptions/${AZ_SUBSCRIPTION_ID}/resourceGroups/${AZ_RESOURCE_GROUP}/providers/Microsoft.ApiManagement/service/${APIM_SERVICE_NAME}/subscriptions/master/listSecrets?api-version=2019-12-01"

# Export the Key to a variable
$ export AZ_API_SUB_KEY="your-api-subscription-key"

# Envoke the echo service
$ curl -i -X POST -d '{ "message": "hello" }' \
     -H "Content-Type: application/json" \
     -H "Ocp-Apim-Subscription-Key: ${AZ_API_SUB_KEY}" \
     -H "Ocp-Apim-Trace: true" \
     "http://${GATEWAY_IP}/echo"

$ kubectl logs -l app=echo-service -c service

################
# If components were updated after deploying, require restart
$ kubectl rollout restart deployment/event-subscriber
$ kubectl rollout status deployment/event-subscriber

$ kubectl rollout restart deployment/demo-apim-gateway
$ kubectl rollout status deployment/demo-apim-gateway

# Ensure components were registered correctly
$ kubectl logs -l app=demo-apim-gateway -c daprd --tail=200



############# NUKE THE BUILD #############
$ kubectl delete -f k8s/gateway.yaml
$ kubectl delete secret demo-apim-gateway-token
$ kubectl delete configmap demo-apim-gateway-env

$ kubectl delete -f k8s/echo-service.yaml
$ kubectl delete -f k8s/event-subscriber.yaml

$ az apim delete --name $APIM_SERVICE_NAME --no-wait --yes

