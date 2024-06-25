# SIM Swap Python code samples

The following code shows, for didactic purposes, a hypothetical Python SDK from a generic Open Gateway's channel partner, also known as an aggregator. The final implementation will depend on the channel partner's development tools offering. Note that channel partners' Open Gateway SDKs are just code modules wrapping authentication and API calls, providing an interface in your app's programming for convenience.

Sample code on how to implement the same flow without an SDK, but with HTTP requests instead, is also provided.

### Table of contents
- [Backend flow](#backend-flow)
  - [Authentication](#authentication)
  - [API usage](#api-usage)
- [Frontend flow](#frontend-flow)
  - [Authentication](#authentication-1)
    - [Requesting the authorization code from the frontend](#requesting-the-authorization-code-from-the-frontend)
    - [Getting the access token from the callback endpoint at the backend](#getting-the-access-token-from-the-callback-endpoint-at-the-backend)
  - [API usage](#api-usage-1)

## Code samples

### Backend flow

The first step is to instantiate the SIM Swap service class included in the corresponding SDK. By providing your app's credentials to the class constructor, it handles the CIBA authentication on its behalf. Providing the phone number as well, as an identifier of the line to be checked for SIM swaps, allows authorization to be 3-legged and enables end-user consent management, and will let your app effectively use the API in a single line of code below.

If not using an SDK, the app will need to follow the two steps of the authentication flow to get an access token and then use it to call the service API by performing HTTP requests.

#### Authentication

Since Open Gateway authentication is 3-legged, meaning it identifies the application, the operator, and the operator's subscriber, who is also the end-user holder of the mobile line, each check for a different phone number needs its own SDK class instantiation, or access token if not using an SDK.

Also note that when using an SDK, each service class will implicitly be aware of the proper service API purpose to use. Otherwise, in case you are not using an SDK, you will need to know the API purpose, as displayed by the channel partner on the API product discovery and ordering process, and provide it in the authentication request, as shown in the samples below.

###### Using a channel partner's SDK
```Python
from aggregator.opengateway_sdk import ClientCredentials, SIMSwap

credentials = ClientCredentials(
  client_id='my-app-id',
  client_secret='my-app-secret'
)

CUSTOMER_PHONE_NUMBER = '+34555555555'

simswap_client = SIMSwap(credentials, CUSTOMER_PHONE_NUMBER)
```

###### Performing HTTP requests (without SDK)
```Python
import requests

# First step:
# Perform an authorization request

customer_phone_number = "+34555555555"

client_id = "my-app-id"
client_secret = "my-app-secret"
my_app_credentials = f"{client_id}:{client_secret}"
app_credentials = base64.b64encode(my_app_credentials.encode('utf-8')).decode('utf-8')
api_purpose = "dpv:FraudPreventionAndDetection#sim-swap"

headers = {
  "Content-Type": "application/x-www-form-urlencoded",
  "Authorization": f"Basic {app_credentials}"
}

data = {
  "login_hint": f"phone_number:{customer_phone_number}",
  "purpose": api_purpose
}

response = requests.post(
  "https://opengateway.aggregator.com/bc-authorize",
  headers=headers,
  data=data
)

auth_req_id = response.json().get("auth_req_id")

# Second step:
# Requesting an access token with the auth_req_id included in the result above

headers = {
  "Content-Type": "application/x-www-form-urlencoded",
  "Authorization": f"Basic {app_credentials}"
}

data = {
  "grant_type": "urn:openid:params:grant-type:ciba",
  "auth_req_id": auth_req_id
}

response = requests.post(
  "https://opengateway.aggregator.com/token",
  headers=headers,
  data=data
)

access_token = response.json().get("access_token")
```

#### API usage

Once your app is authenticated, it only takes a single line of code to use the service API and effectively get a result.

###### Using a channel partner's SDK
```Python
result = simswap_client.retrieve_date()

print(f"SIM was swapped: {result.strftime('%B %d, %Y, %I:%M:%S %p')}")
```

###### Performing HTTP requests (without SDK)
```Python
headers = {
  "Content-Type": "application/json",
  "Authorization": f"Bearer {access_token}"
}

data = {
  "phoneNumber": customer_phone_number
}

response = requests.post(
  "https://opengateway.aggregator.com/sim-swap/v0/retrieve-date",
  headers=headers,
  json=data
)

result = response.json().get("latestSimChange")

print(f"SIM was swapped: {result.strftime('%B %d, %Y, %I:%M:%S %p')}")
```

### Frontend flow

Although it is not common to face a use case for SIM Swap where the user device is involved in the flow and, at the same time, it is easier to get the phone number on the frontend than on the backend, or it is more precise in terms of data quality on each end, if you wanted to start the service API consumption from a frontend application, you would need to implement the OIDC's Authorization Code Flow instead of CIBA. This flow implies your application providing a callback URL that you will need to publish online hosted on your backend server, and in which your application's backend will get a `code` authorizing it to use the Open Gateway APIs for your end-user.

This flow allows the mobile network operator to effectively identify the user by resolving the IP address of their device, running your application, by getting an HTTP redirection and returning a `code` that will reach out to your callback URL. You can check the Camara documentation on the Authorization Code Flow [here](https://github.com/camaraproject/IdentityAndConsentManagement/blob/release-0.1.0/documentation/CAMARA-API-access-and-user-consent.md#authorization-code-flow-frontend-flow).

From this point, your application, from its backend, will request an access token and use it to call the service API.

#### Authentication

Some Open Gateway's channel partners may provide you with frontend SDKs to handle the Authorization Code Flow on your application's behalf, providing some extra features like checking your end-user's device network interface to avoid problems caused by it being connected to a WiFi network, for instance, since this flow relies on the mobile line connectivity for the operator to identify the end-user and authorize your application.

The following samples show how to implement the flow without a frontend SDK with these premises:
- Application's frontend performs an HTTP request to get a `code` and provides a `redirect_uri` it wants such `code` to be redirected to.
- Application's frontend will receive an HTTP redirect (status 302) and needs to be able to handle it. If it is a web application running on a web browser, the browser will natively follow the redirection. If it is not, in depends on the coding language and/or HTTP module or library used, or on its settings, how the flow will follow all the way to your application's backend through the mobile network operator authentication server.
- Application's backend receives the `code` from this HTTP redirection, by publishing an endpoint in the given `redirect_uri`, and then exchanges it for an access token. The latter can be achieved by using a backend SDK as shown in the [Backend flow](#backend-flow).

##### Requesting the authorization code from the frontend

The following samples show how your application can trigger the authentication flow from the frontend:

```HTML
<!--
    In this example you need your callback URL to continue the flow calling the service API
    and providing an HTML response to be shown in the web browser displaying the result.
    The following form is published from the same web server hosting the callback URL.
-->

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>API Request Form</title>
</head>
<body>
    <h1>SIM swap date check</h1>
    <form id="apiRequestForm" action="https://opengateway.aggregator.com/authorize" method="GET">
        <input type="hidden" name="client_id" value="my-app-id">
        <input type="hidden" name="response_type" value="code">
        <input type="hidden" name="purpose" value="dpv:FraudPreventionAndDetection#sim-swap">
        <input type="hidden" name="redirect_uri" value="/simswap-callback">
        
        <label for="state">Your phone number:</label>
        <input type="text" id="state" name="state" value="+34555555555" required>
        
        <button type="submit">Check</button>
    </form>
</body>
</html>
```

##### Getting the access token from the callback endpoint at the backend

Samples represent how to publish the callback URL in Python, so the code from the Auth Code Flow can be received front door:

###### Using a channel partner's SDK
```Python
from aggregator.opengateway_sdk import ClientCredentials, SIMSwap
from flask import Flask, request

credentials = ClientCredentials(
  client_id='my-app-id',
  client_secret='my-app-secret'
)

app = Flask(__name__)

@app.route('/simswap-callback', methods=['GET'])
def simswap_callback():
  code = request.args.get('code')
  phoneNumber = request.args.get('state')
  simswap_client = SIMSwap(credentials, auth_code=code)

if __name__ == '__main__':
  app.run()
```

###### Performing HTTP requests (without SDK)

This step is pretty much the same that in the case of the [backend flow](#performing-http-requests-without-sdk), but using the `code` received from the frontend on the callback URL. Instead of the using CIBA's grant type and the `auth_req_id` use the following:
```Python
import requests

payload = {
  "grant_type": "authorization_code",
  "code": code,
  "redirect_uri": your_backend_callback_url
}

headers = {
  "Content-Type": "application/x-www-form-urlencoded"
}

response = requests.post(
  "https://opengateway.aggregator.com/token",
  data=payload,
  headers=headers
)

result = response.json()
accessToken = result.get("access_token")
```

#### API usage

Once your app is authenticated it only takes a single line of code to use the service API and effectively get a result.

###### Using a channel partner's SDK
```Python
result = await simswap_client.retrieve_date(phone_number)

print(f"SIM was swapped: {result.strftime('%B %d, %Y, %I:%M:%S %p')}")
```

###### Performing HTTP requests (without SDK)

Once you app has gotten an access token, the way to call the service API is exactly the same as shown in the [backend flow](#performing-http-requests-without-sdk-1).