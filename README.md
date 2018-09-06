# GitHub OpenID Connect Wrapper

[![Build Status](https://travis-ci.org/TimothyJones/github-openid-wrapper.svg?branch=master)](https://travis-ci.org/TimothyJones/github-openid-wrapper)
[![Maintainability](https://api.codeclimate.com/v1/badges/f787719be529b1c0e8ee/maintainability)](https://codeclimate.com/github/TimothyJones/github-openid-wrapper/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/f787719be529b1c0e8ee/test_coverage)](https://codeclimate.com/github/TimothyJones/github-openid-wrapper/test_coverage)

Do you want to add GitHub as an OIDC (OpenID Connect) provider to an AWS Cognito User Pool? Have you run in to trouble because GitHub only provides OAuth2.0 endpoints, and doesn't support OpenID Connect?

This project allows you to wrap your GitHub OAuth App in an OpenID Connect layer, allowing you to use it with AWS Cognito.

Here are some questions you may immediately have:

* **Why does Cognito not support federation with OAuth?** Because OAuth provides
  no standard way of requesting user identity data. (see the [background](#background)
  section below for more details).

* **Why has no one written a shim to wrap general OAuth implementations with an
  OpenID Connect layer?** Because OAuth provides no standard way of requesting
  user identity data, any shim must be custom written for the particular OAuth
  implementation that's wrapped.

* **GitHub is very popular, has someone written this specific custom wrapper
  before?** As far as I can tell, if it has been written, it has not been open
  sourced. Until now!

## Project overview

When deployed, this project sits between Cognito and GitHub:

![Overview](docs/overview.png)

This allows you to use GitHub as an OpenID Identity Provider (IdP) for federation with a Cognito User Pool.

The project implements everything needed by the [OIDC User Pool IdP authentication flow](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-flow.html) used by Cognito.

It implements the following endpoints from the
[OpenID Connect Core Spec](https://openid.net/specs/openid-connect-core-1_0.html):

* Authorization - used to start the authorisation process ([spec](https://openid.net/specs/openid-connect-core-1_0.html#AuthorizationEndpoint))
* Token - used to exchange an authorisation code for an access and ID token ([spec](https://openid.net/specs/openid-connect-core-1_0.html#TokenEndpoint))
* UserInfo - used to exchange an access token for information about the user ([spec](https://openid.net/specs/openid-connect-core-1_0.html#UserInfo))
* jwks - used to describe the keys used to sign ID tokens ([implied by spec](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderMetadata))

It also implements the following [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) endpoint:

* Configuration - used to discover configuration of this OpenID implementation's
  endpoints and capabilities. ([spec](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig))

### Deployment

This project is intended to be deployed as a series of lambda functions alongside
an API Gateway. This means it's easy to use in conjunction with Cognito, and
should be cheap to host and run.

## Getting Started

You will need to:

* Create a Cognito User Pool ([instructions](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-as-user-directory.html)).
* Configure App Integration for your User Pool ([instructions](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-configuring-app-integration.html)). Note down the domain name.
* Create a GitHub OAuth App ([instructions](https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/), with the following settings:
  * Authorization callback URL: `https://<Your Cognito Domain>/oauth2/idpresponse`
  * Note down the Client ID and secret

### Deployment with lambda and API Gateway.


* Install the `aws` and `sam` CLIs from AWS:
  * `aws` ([install instructions](https://docs.aws.amazon.com/cli/latest/userguide/installing.html)) and configured
  * `sam` ([install instructions](https://docs.aws.amazon.com/lambda/latest/dg/sam-cli-requirements.html))

* Run `aws configure` and set appropriate access keys etc
* Set environment variables for the OAuth App client/secret, callback url, stack name, etc:

       cp example-config.sh config.sh
       vim config.sh # Or whatever your favourite editor is

* Run `npm install` and `npm run deploy`
* Note down the DNS of the deployed API Gateway (available in the AWS console).
* Configure the OIDC integration in AWS console for Cognito (described below, but following [these instructions](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-idp.html)). The following settings are required:
    * Client ID: The GitHub Client ID above
    * Authorize scope: `openid read:user user:email`
    * Issuer: `https://<Your API Gateway DNS name>/Prod`
    * For some reason, Cognito is unable to use API gateway to do OpenID Discovery. You will need to configure the endpoints manually. They are:
        * Authorization endpoint: `https://<Your API Gateway DNS name>/Prod/authorize`
        * Token endpoint: `https://<Your API Gateway DNS name>/Prod/token`
        * Userinfo endpoint: `https://<Your API Gateway DNS name>/Prod/userinfo`
        * JWKS uri: `https://<Your API Gateway DNS name>/Prod/jwks.json`
* Configure the Attribute Mapping in the AWS console:

![Attribute mapping](docs/attribute-mapping.png)


That's it! If you need to redeploy, all you need to do is run `npm run deploy` again.

## The details

### Background

There are two important concepts for identity federation:

- Authentication: Is this user who they say they are?
- Authorisation: Is the user authorised to use a particular resource?

#### OAuth

[OAuth2.0](https://tools.ietf.org/html/rfc6749) is an _authorisation_ framework,
used for determining whether a user is allowed to access a resource (like
private user profile data). In order to do this, it's usually necessary for
_authentication_ of the user to happen before authorisation.

This means that most OAuth2.0 implementations (including GitHub) [include authentication in a step of the authorisation process](https://medium.com/@darutk/new-architecture-of-oauth-2-0-and-openid-connect-implementation-18f408f9338d).
For all practical purposes, most OAuth2.0 implementations (including GitHub)can
be thought of as providing both authorisation and authentication.

Below is a diagram of the authentication code flow for OAuth:

![OAuth flow](docs/oauth-flow.svg)

(The solid lines are http requests from the browser, and then dashed lines are
back-channel requests).

As you can see in the diagram, a drawback of OAuth is that it provides no
standard way of finding out user data such as name, avatar picture, email
address(es), etc. This is one of the problems that is solved by OpenID.

#### OpenID Connect

To provide a standard way of learning about users,
[OpenID Connect](https://openid.net/connect/) is an identity layer built on top
of OAuth2.0. It extends the `token` endpoint from OAuth to include an ID Token
alongside the access token, and provides a `userinfo` endpoint, where information
describing the authenticated user can be accessed.

![OpenID Connect](docs/openid-flow.svg)

OpenID Connect describes a standard way to get user data, and is therefore a good choice
for identity federation.

### A custom shim for GitHub

This project provides the OpenID shim to wrap GitHub's OAuth implementation, by combining the two
diagrams:

![GitHub Shim](docs/shim.svg)

The userinfo request is handled by joining two GitHub API requests: `/user` and `/user/emails`.

You can compare this workflow to the documented Cognito workflow [here](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-flow.html)

#### Code layout

    ├── scripts         # Bash scripts for deployment and private key generation
    ├── src             # Source code
    │    ├── __mocks__  # Mock private key data for tests
    │    └── lambda     # AWS lambda handlers
    │         └── util  # Non-handler code for lambdas
    ├── docs            # Documentation images
    ├── config          # Configuration for tests
    └── dist            # Where `npm run build-dist` saves output

#### npm targets

- `build` and `build-dist`: create packages in the `dist` folder for the lambda
  deployment
- `test`: Run unit tests with Jest
- `lint`: Run `eslint` to check code style
- `test-dev`: Run unit tests continuously, watching the file system for changes
  (useful for development)
- `deploy`: This script builds the project, then creates and deploys the
  cloudformation stack with the API gateway and the endpoints as lambdas

#### Scripts

- `scripts/create-key.sh`: If the private key is missing, generate a new one.
  This is run as a preinstall script before `npm install`
- `scripts/deploy.sh`: This is the deploy part of `npm run deploy`. It uploads
  the dist folder to S3, and then creates the cloudformation stack that contains  
  the API gateway and lambdas

#### Tests

Tests are provided with [Jest](https://jestjs.io/) using
[`chai`'s `expect`](http://www.chaijs.com/api/bdd/), included by a shim based on [this blog post](https://medium.com/@RubenOostinga/combining-chai-and-jest-matchers-d12d1ffd0303).

[Pact](http://pact.io) consumer tests for the GitHub API connection are provided
in `src/github.pact.test.js`. There is currently no provider validation performed.

#### Private key

The private key used to make ID tokens is stored in `./jwtRS256.key` once
`scripts/create-key.sh` is run (either manually, or as part of `npm install`).
You may optionally replace it with your own key - if you do this, you will need
to redeploy.

#### Missing features

This is a near-complete implementation of [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html).
However, since the focus was on enabling Cognito's [authentication flow](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-flow.html),
you may run in to some missing features if you wish to use it with a different
client.

**Missing Connect Core Features:**

* Private key rotation ([spec](https://openid.net/specs/openid-connect-core-1_0.html#RotateSigKeys))
* Refresh tokens ([spec](https://openid.net/specs/openid-connect-core-1_0.html#RefreshTokens))
* Passing request parameters as JWTs ([spec](https://openid.net/specs/openid-connect-core-1_0.html#JWTRequests))

If you don't know what these things are, you are probably ok to use this project.

**Missing non-core features:**

A full OpenID implementation would also include:

* [The Dynamic client registration spec](https://openid.net/specs/openid-connect-registration-1_0.html)
* The [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) endpoints beyond `openid-configuration`

**Known issues**

For some reason, Cognito won't use the discovery endpoint if deployed via lambda. However, the four other endpoints can be specified manually.

## Extending

This section contains pointers if you would like to extend this shim.

### Using other OAuth providers

If you want to use a provider other than GitHub, you'll need to change the contents of `userinfo` in `src/openid.js`.

### Using a custom GitHub location

If you're using an on-site GitHub install, you will need to change the API
endpoints used when the `github` object is initialised.

### Including additional user information

If you want to include custom claims based on other GitHub data,
you can extend `userinfo` in `src/openid.js`. You may need to add extra API
client calls in `src/github.js`


## Contributing

Contributions are welcome, especially for the missing features! Pull requests and issues are very welcome.

## License

[BSD 3-Clause License](LICENSE)
