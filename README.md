# Introduction
This repository contains a sample implementation for an API Connector for the Talentech Evaluation API using ASP.NET Core. Run the project locally in order to view OpenAPI docs, which are served at the default route.

The API Connector is the component that connects your API with the Talentech ATSes via the Talentech Evaluation API as shown in the high level illustration below.

![Illustration of components](docs/images/components.png)

Key terms
----------
- Evaluation API - The Talentech app partners will integrate with
- Partner App - The existing app managed by the partner.
- API Connector - A facade API controlled by the Partner. This is what will connect the Evaluation API and the Partner App's APIs. 
- Invitations - An invitation for a candidate. This will be sent from the Evaluation API to the API connector
- Results - Results after a candidate has completed an assessment test or reference check. This will be posted from the API connector and back to the Evaluation API
- ApiBaseUrl - The base url of the Api Connector. This is used to resolve the endpoints the Evaluation API expects the API connector to make available.

# What is needed to integrate
The endpoints below must be implemented by a partner in order to integrate with the Talentech Evaluation API. 
- Invitations endpoint
- Health check endpoint
- Evaluations endpoint
- Authorization endpoints (See the OAuth section below)

The relative URLs implemented by the partners can be found here:
https://github.com/Talentech/EvaluationApiSampleIntegration/blob/master/src/Config/Constants.cs

The partner should post results back to the EvaluationApi. In order to do this, an access token must be retrieved from a token server managed by Talentech. This token should be included in the authorization header with the api call to the EvaluationApi. A sample implementation of how this can be done in dotnet core can be found here:
https://github.com/Talentech/EvaluationApiSampleIntegration/blob/master/src/Services/Clients/EvaluationApi/EvaluationApiClient.cs

# OAuth
The OAuth2.0 authorization_code grant is supported by the Evaluation API and is a mandatory requirement for partners who wish to integrate. 

The Api connector should implement the following endpoints in the API connector:
- AuthorizationEndpoint 
- TokenEndpoint
- DeauthorizationEndpoint

These will typically be simple proxy endpoints that route the request on to a backend OAuth service managed by the partner. 

The workflow for granting the Evaluation API access to the PartnerApp's resources works as follows:

- When an ATS customer subscribes to a partner integration, we will need to get their authorization details in order to access their data in the Partner's API on their behalf. 
- This is done by redirecting an admin user from the customer to the Authorization endpoint (which is managed by the Partner).
- At the authorization endpoint, the customer's user will authenticate themselves, as well as grant the Talentech app the required permissions.
- Once done, the user must be redirected back to the EvaluationApi's redirect_uri. An authorization code should be included as a query string parameter.
- When the user hits the Evaluation API's redirect_uri endpoint, an API call will be made to exchange the authorization code for a token.

The {evaluation_api_redirect_uri} is a parameter that should be validated by the partner api. The URL is as follows:
https://evaluation.talentech.io/Customers/OAuth/Callback

We currently support the following token types: "Refresh token" and "Fixed token". 
- Refresh tokens are tokens we will use to retrieve new access tokens from the partner if the existing access token is about to expire.
- Fixed tokens are permanent tokens that will be stored and used as-is. An example of this would be an API key. 

Please note that the samples in the repository currently implement the Fixed token mode.

# Error handling
Whenever an API call to the Api connector returns an HTTP error code, the EvaluationApi will try to deserialize the HTTP content to a given format. The error objects have a type parameter. 
- SystemErrors are meant for the Evaluation API to use internally.
- UserErrors may be shown to end users.

# Retry policies
When results are posted back to Talentech by the partner, the Evaluation API will act as a gateway and forward the data to the ATS that sent the invitation. As the API calls may occasionally fail, it's important that the partner implements retry policies to ensure that the customer eventually receives their data. The retry policy should include an exponential backoff to prevent overloading the API.

# Sequence diagrams
Below is a description of the flows we support. The one we'll use for a given partner's app is governed by the token type (Fixed or Refresh token) configured for the app in question.



![Sequence diagrams](docs/images/SequenceDiagrams.png)
