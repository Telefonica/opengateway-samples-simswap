# Open Gateway's SIM Swap API sample app

The following samples show how to use the [Open Gateway SIM Swap API](https://opengateway.telefonica.com/en/apis/sim-swap), for fraud prevention purposes, in order to check if a given mobile line, identified by its phone number, has gotten a SIM card swap lately.

[Go to the sample app](#sample-app)

## Sample code

The following code snippets show how to use the SIM Swap API in different programming languages, with our without a channel partner's SDK, and on different scenarios triggered from the end-user device or from a backend. Channel partner's are hyperscalers and aggregators involved in the Open Gateway initiative to provide application owners with a single API exposure platform that routes API calls to the telcos that end-users are subscribers to.

Sample code on how to consume the API without an SDK, directly with HTTP requests, is also provided, and it is common and valid no matter what your partner is, thanks to the Camara standardization. If you do not use an SDK you need to code the HTTP calls and additional stuff like encoding your credentials, calling authorization endpoints, handling tokens, etc. You can check our sample [Postman collection](https://bxbucket.blob.core.windows.net/bxbucket/opengateway-web/uploads/OpenGateway.postman_collection.json) as a reference.

Most likely, this API will be consumed in a backend flow, since it is the application owner not the end-user who wants to take advantage of its functionality. The authentication protocol used in Open Gateway for backend flows is the OIDC standard CIBA (Client Initiated Backchannel Authentication). You can check the Camara documentation on this flow [here](https://github.com/camaraproject/IdentityAndConsentManagement/blob/release-0.1.0/documentation/CAMARA-API-access-and-user-consent.md#ciba-flow-backend-flow).

The code snippets are just examples and may need to be adapted to your specific use case:
- [Node.js](./docs/code_snippets/node.md)
- Python
- Java

## Sample app

In addition to the sample codes above, you can also check our SIM Swap's sample app, which is an open source comprehensive backend application, coded in Python, consuming the Open Gateway's SIM Swap API for a simple fraud prevention use case:

[SIM Swap sample Python backend application](https://github.com/Telefonica/opengateway-samples-simswap-backend)

This sample app uses the Telefónica's Open Gateway Sandbox and its SDK so you don't have to subscribe an actual channel partner to test the API. In order to have access to our Sandbox, you need to register in one of our programs:
- [Partner Program](https://opengateway.telefonica.com/en/partner-program)
- [Developer Hub](https://opengateway.telefonica.com/en/developer-hub)

Once you are onboarded, follow these steps:
1. Access the Sandbox and register your application, you will credentials for it to use Sandbox's SDK and/or API gatewway: a client_id and a client_secret
2. Clone the sample app repository, and follow the instructions in the README file to setup your app credentials and run the app
3. Test the app with any phone number for mock responses, or with a real mobile line you own and have asked to be added to the whitelist, if you want to test the API on one of our actual operators
