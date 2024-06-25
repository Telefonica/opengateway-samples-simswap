# SIM Swap Node.js code samples

The following code shows, for didactic purposes, an hypothetical Node.js SDK from a generic Open Gateway's channel partner, also known as aggregator. The final implementation will depend on the channel partner's development tools offering. Note that channel partners' Open Gateway SDKs are just code modules wrapping authentication and API calls providing an interface in your app's programming for convenience.

Sample code on how to implement the same flow without and SDK, but with HTTP requests instead, is also provided.

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

First step is to instantiate the SIM Swap service class included in the corresponding SDK. By providing your app's credentials to the class constructor, it handles the CIBA authentication on its behalf. Providing the phone number as well, as an identifier of the line to be checked for SIM swaps, allows authorization to be 3-legged and enables end-user consent management, and will let your app to just effectively use the API in a single line of code below.

If not using an SDK, the app will need to follow the two steps of the authentication flow to get an access token, an then use it to call the service API, by performing HTTP requests.

#### Authentication

Since Open Gateway authentication is 3-legged, meaning it identifies the application, the operator and the operator's subscriber, who is also the end-user holder of the mobile line, each check for a different phone number needs its own SDK class instantiation, or access token if not using an SDK.

Also note that when using an SDK, each service class will implicitly be aware of the proper service API purpose to use. Otherwise, in case your are not using an SDK, you will need to know the API purpose, as displayed by the channel partner on the API product discovery and ordering process, and provide it in the authentication request, as shown in the samples below. 

###### Using a channel partner's SDK
```Node
import { ClientCredentials, SIMSwap } from "aggregator/opengateway-sdk"

const credentials: ClientCredentials(
    clientId: 'my-app-id',
    clientSecret: 'my-app-secret'
)

const CUSTOMER_PHONE_NUMBER = '+34555555555'

const simswapClient = new SIMSwap(credentials, null, CUSTOMER_PHONE_NUMBER)
```

###### Performing HTTP requests (without SDK)
```JavaScript
// First step:
// Perform an authorization request

let customerPhoneNumber = "+34555555555";

let clientId = "my-app-id";
let clientSecret = "my-app-secret";
let appCredentials = btoa(`${clientId}:${clientSecret}`);
let apiPurpose = "dpv:FraudPreventionAndDetection#sim-swap";

const myHeaders = new Headers();
myHeaders.append("Content-Type", "application/x-www-form-urlencoded");
myHeaders.append("Authorization", `Basic ${appCredentials}`);

const urlencoded = new URLSearchParams();
urlencoded.append("login_hint", `phone_number:${customerPhoneNumber}`);
urlencoded.append("purpose", apiPurpose);

const requestOptions = {
  method: "POST",
  headers: myHeaders,
  body: urlencoded
};

let authReqId;

fetch("https://opengateway.aggregator.com/bc-authorize", requestOptions)
  .then(response => response.json())
  .then(result => {
    authReqId = result.auth_req_id;
  })

// Second step:
// Requesting an access token with the auth_req_id included in the result above

const myHeaders = new Headers();
myHeaders.append("Content-Type", "application/x-www-form-urlencoded");
myHeaders.append("Authorization", `Basic ${appCredentials}`);

const urlencoded = new URLSearchParams();
urlencoded.append("grant_type", "urn:openid:params:grant-type:ciba");
urlencoded.append("auth_req_id", authReqId);

const requestOptions = {
  method: "POST",
  headers: myHeaders,
  body: urlencoded
};

let accessToken;

fetch("https://opengateway.aggregator.com/token", requestOptions)
  .then(response => response.json())
  .then(result => {
    accessToken = result.access_token;
  })
```

#### API usage

Once your app is authenticated it only takes a single line of code to use the service API and effectively get a result.

###### Using a channel partner's SDK
```Node
let result = await simswapClient.retrieveDate()

console.log(`SIM was swapped: ${result.toLocaleString('en-GB', { timeZone: 'UTC' })}`)
```

###### Performing HTTP requests (without SDK)
```JavaScript
const myHeaders = new Headers();
myHeaders.append("Content-Type", "application/json");
myHeaders.append("Authorization", `Bearer ${accessToken}`);

const requestBody = JSON.stringify({
  "phoneNumber": customerPhoneNumber
});

const requestOptions = {
  method: "POST",
  headers: myHeaders,
  body: requestBody
};

fetch("https://opengateway.aggregator.com/sim-swap/v0/retrieve-date", requestOptions)
  .then(response => response.json())
  .then(result => {
    console.log(`SIM was swapped: ${result.latestSimChange}`)
  })
```

### Frontend flow

Although it is not common to face a use case for SIM Swap where the user device is involved in the flow and, at the same time, it is easier to get the phone number on the frontend that on the backend, or it is more precise in terms of data quality on each end, if you wanted to start the service API consumption from a frontend application, you would need to implement the OIDC's Authorization Code Flow instead of CIBA. This flow implies your application providing a callback URL that you will need to publish online hosted on your backend server, and in which your application's backend will get a `code` authorizing it to use the Open Gateway APIs for your end-user.

This flow allows the mobile network operator to effectively identify the user by resolving the IP address of their device, running your application, by getting an HTTP redirection and returning a `code` that will reach out to your callback URL. You can check the CAMARA documentation on the Authorization Code Flow [here](https://github.com/camaraproject/IdentityAndConsentManagement/blob/release-0.1.0/documentation/CAMARA-API-access-and-user-consent.md#authorization-code-flow-frontend-flow).

From this point, your application, from its backend, will request an access token, and use it to call the service API.

#### Authentication

Some Open Gateway's channel partners may provide you with frontend SDKs to handle the Authorization Code Flow on your application's behalf, providing some extra features like checking your end-user's device network interface to avoid problems caused from it being connected to a WiFi network, for instance, since this flow relies on the mobile line connectivity for the operator to identify the end-user and authorize your application.

The following samples show how to implement the flow without a frontend SDK with these premises:
* Application's frontend performs an HTTP request to get a `code`, and provides a `redirect_uri` it wants such `code` to be redirected to.
* Application's frontend will receive an HTTP redirect (status 302) and needs to be able to handle it. If it is a web application running on a web browser, the browser will natively follow the redirection. If it is not, in depends on the coding language and/or HTTP module or library used, or on its settings, how the flow will follow all the way to your application's backend through the mobile network operator authentication server.
* Application's backend receives the `code` from this HTTP redirection, by publishing an endpoint in the given `redirect_uri`, and then exchanges it for an access token. The latter can be achieved by using a backend SDK as shown in the [Backend flow](#backend-flow).

##### Requesting the authorization code from the frontend

The following samples show how your application can trigger the authentication flow from the frontend either from code or by submitting a simple HTML form. The same can be achieved from code in any other programming language with the ability to perform HTTP requests:

###### From JavaScript code
```JavaScript
let userPhoneNumber = "+34555555555";

let clientId = "my-app-id";
let clientSecret = "my-app-secret";
let apiPurpose = "dpv:FraudPreventionAndDetection#sim-swap";
let myCallbackEndpoint = "https://my_app_server/simswap-callback";

const params = {
  client_id: clientId,
  response_type: "code",
  purpose: apiPurpose,
  redirect_uri: myCallbackEndpoint,
  state: userPhoneNumber // Using `state` as the parameter that is always forwarded to the redirect_uri
};

// Create the query string
const queryString = new URLSearchParams(params).toString();

// URL with query string
const url = `https://opengateway.aggregator.com/authorize?${queryString}`;

const requestOptions = {
  method: "GET",
  redirect: "follow"
};

fetch(url, requestOptions);
```

###### From an HTML form
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

Samples represent how to publish the callback URL in Node.js, so the code from the Auth Code Flow can be received front door:

###### Using a channel partner's SDK
```Node
import { ClientCredentials, SIMSwap } from "aggregator/opengateway-sdk"
import express from "express"

const credentials: ClientCredentials(
    clientId: 'my-app-id',
    clientSecret: 'my-app-secret'
)

const app = express()
const port = 3000

app.get('/simswap-callback', (req, res) => {
    const code = req.query.code
    const phoneNumber = req.query.state
    const simswapClient = new SIMSwap(credentials, code)
})

app.listen(port, () => {
    console.log(`SIM Swap callback URL is running`);
})
```

###### Performing HTTP requests (without SDK)

This step is pretty much the same that in the case of the [backend flow](#performing-http-requests-without-sdk), but using the `code` received from the frontend on the callback URL. Instead of the using CIBA's grant type and the `auth_req_id` use the following:

```JavaScript
const urlencoded = new URLSearchParams();
urlencoded.append("grant_type", "authorization_code");
urlencoded.append("code", code);
urlencoded.append("redirect_uri", yourBackendCallbackUrl);

const requestOptions = {
  method: "POST",
  headers: myHeaders,
  body: urlencoded
};

let accessToken;

fetch("https://opengateway.aggregator.com/token", requestOptions)
  .then(response => response.json())
  .then(result => {
    accessToken = result.access_token;
  })
```

#### API usage

Once your app is authenticated it only takes a single line of code to use the service API and effectively get a result.

###### Using a channel partner's SDK
```Node
let result = await simswapClient.retrieveDate(phoneNumber)

console.log(`SIM was swapped: ${result.toLocaleString('en-GB', { timeZone: 'UTC' })}`)
```

###### Performing HTTP requests (without SDK)

Once you app has gotten an access token, the way to call the service API is exactly the same as shown in the [backend flow](#performing-http-requests-without-sdk-1).