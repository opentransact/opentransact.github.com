---
title: Crowdfunding
layout: default
---

# Crowdfunding

Crowdfunding is when a group of people pool their money together to fund a project. The most popular example right now is [Kickstarter](http://kickstarter.com).

<div class="thumbnail"><a href="http://www.kickstarter.com/projects/1202515667/the-city-troll-a-graphic-novel?ref=spotlight"><img src="https://img.skitch.com/20120111-2tc6mnte63qr52sahpu1xsbqk.preview.jpg" alt="A KickStarter project" /></a></div>

To avoid fraud and ensure the success of the project a popular implementation rule is to have a minimum funding goal and an expiration date on the project.

Eg. Project X is looking for minimum $10,000 within the next month.

A crowdfunding service such as [Kickstarter](http://kickstarter.com) would handle most of the business logic surrounding crowdfunding.

They can integrate with existing OpenTransact providers for the funding part as well as present an OpenTransact interface for each project as well.

We will handle both cases here:

- [The Crowd Funding service using OpenTransact as a Client](#client)
- [The Crowd Funding service as the OpenTransact provider](#provider)

## Roles

- Offerer - The person/organization listing a project for funding (Alice)
- Supporter - The person wanting to help fund project (Bob)
- Crowdfunding Service - The service provider (CrowdStarter)
- Payment Provider - The payment provider used (PayMe)

## <span id="client">Integrate with OpenTransact payment provider</span>

1. Alice registers her project on Crowdstarter site and decides she needs $10,000
2. CrowdStarter approves the project and sets an expiration date 2 months in the future
3. Bob views Alice's project on CrowdStarter's site.
4. <span id="pledge">Bob clicks the Pledge button and enters an amount ($20) he wants to pledge.</span>
5. The Crowdfunding site generates an [OpenTransact Transfer Authorization](/core.html#transfer-authorization-1)  link to Bob's payment provider PayMe.
6. Bob clicks the link and goes to PayMe
7. PayMe asks Bob if he wants to approve the request for $20 valid until 2 months in the future.
8. Bob approves and gets redirected back to CrowdStarter with an [OAuth2.0 authorization code](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-4.1.2).
9. CrowdStarter exchanges the authorization code with a [OAuth 2.0 access token request](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-4.1.3) and stores the resulting token in their database

Now 2 things can happen:

### Project meets it's funding goal

Cool everyone is happy.

CrowdStarter finds all successful pledges to the project and goes through each one performing the following:

1. CrowdStarter performs an [OpenTransact Transfer](http://www.opentransact.org/core.html#transfer-1) with the Access Token from Step. 9 above
2. CrowdStarter funds using to Alice's account at PayMe again using an [OpenTransact Transfer](http://www.opentransact.org/core.html#transfer-1) but this time with their own Access Token

### Project did not meet it's funding goal

The Access Token received earlier should expire shortly after the date the fundraising session expired, so CrowdStarter doesn't really have to do anything.

However we think it is best practise to explicitly revoke the token using the [OAuth 2.0 Token revocation spec](http://tools.ietf.org/html/draft-lodderstedt-oauth-revocation-03)


### Detailed examination of steps

It helps to see how this actually is implemented. So here we show it using raw HTTP posts. We will create examples in Ruby and other languages later.

#### <span id="transfer_authorization">Transfer Authorization</span>

The Transfer Authorization link in Step 5. is built up with the following parameters:

- amount=20.00 # The amount to authorize
- to=alice@project.example.com # Alice's email address
- for=http://crowdstarter.example.com/projects/123123 # The URL to the project
- validity=P34D # The authorization should be valid for 34 days from now [see](http://www.opentransact.org/core.html#validity-duration)
- client_id=1234134 # The preregistered OAuth client id of CloudStarter on PayMe
- response_type=code # code or token [see](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-1.3)
- redirect_uri=http://crowdstarter.example.com/projects/12313/thanks
- state=123455 # A Unique value used by CrowdStarter to identify the user

These parameters are URL form encoded and appended as a query string to PayMe's OpenTransact URL:

    https://payme.example.com/usd?amount=20&for=http://crowdstarter.example.com/projects/123123&validity=P34D&client_id=1234134&response_type=code&redirect_uri=http://crowdstarter.example.com/projects/12313/thanks&state=123455


#### Fetch Access Token

Once Bob has authorized the transfer and is redirected back to the CrowdStarter site in step 8 he receives the following parameters in the query string [see](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-4.1.2):

- code=4a94c45bc356bcebf76f2d7d2ce435b5 # a unique access code generated by PayMe
- state=123455 # the state variable sent a long before

CrowdStarter now performs an [OAuth 2.0 Token Request](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-4.1.3) to get the AccessToken they will use later.

    POST /token HTTP/1.1
    Host: payme.example.com
    Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
    Content-Type: application/x-www-form-urlencoded;charset=UTF-8

    grant_type=authorization_code&code=4a94c45bc356bcebf76f2d7d2ce435b5&redirect_uri=http://crowdstarter.example.com/projects/12313/thanks

PayMe returns an Access Token as JSON:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "access_token":"2YotnFZFEjr1zCsicMWpAA",
      "token_type":"example",
      "expires_in":2937600
    }

Note that it includes the expires_in parameter which should contain the requested validity duration in seconds.

#### Perform OpenTransact Transfer

Once the payment goal is achieved CrowdStarter performs individual transfers like this.

    POST /usd HTTP/1.1
    Host: payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 0


It uses the access token as received above and sends an empty body. PayMe knows the amount and recipient so no need to specify anything else.

## <span id="provider">The Crowd Fund service as an OpenTransact provider</span>

[Assets](http://www.opentransact.org/core.html#assets) are the most important thing to understand about OpenTransact.

A payment providers service for USD is one example of an asset. Stock in Google is another. Each asset has a unique [Transaction URL](http://www.opentransact.org/core.html#transaction-url) which accepts the various OpenTransact requests.

For a Crowd Funding service like CrowdStarter each project is an asset.

As in each $1 unit in the asset means that I am pledging $1 to support the project.

So lets assume Alice's project's URL is:

    http://crowdstarter.example.com/projects/123123

This is the url which would contain the exact same information on the page as it does today.

So what would enabling OpenTransact on this URL allow people to do?

- Alice could promote specific links (Transfer Requests) in her social network
- Crowd Funding aggregators could be built finding the best projects around the web
- Crowd Funding mobile apps could be easily created supporting multiple providers

So the main two requirements of this is to be able to

- Request a Pledge from another web app/twitter/email
- Perform a Pledge via a third party app

### Listing projects

A simple way for projects to be listed would be to use the [Atom Syndication Format](http://tools.ietf.org/html/rfc4287). Each Crowd Funding service could list their projects with photos/descriptions/media etc.

The item url would be the above OpenTransact URL making it easy for both web apps and mobile apps to learn about the projects and integrate with them.

### OpenTransact Request

[OpenTransact](/) consists of 3 types of requests:

- [Transfer Request](http://www.opentransact.org/core.html#transfer-request)
- [Transfer Authorization](http://www.opentransact.org/core.html#transfer-authorization)
- [Transfer](http://www.opentransact.org/core.html#transfer)

The Transfer Request is used in payment buttons, email links and anywhere you would request that a user pledges to support the project.

Transfer Authorization is a bit more complicated to implement. It requests the same transfer as the Transfer Request but instead of performing it, it issues OAuth 2 credentials to allow the service to perform the transfer later. I can't quite think of a use case where this would be useful for a Crowd Funding service.

The Transfer it self is used by an external app such as a mobile phone app to pledge to support a project.

### Transfer Request

The [OpenTransact parameters](http://www.opentransact.org/core.html#transfer-request-1) relevant to CrowdFunding could be:

- *to* The person making the pledge (see below)
- *amount* The amount I want to pledge
- *note* Textual description, which can include hash tags. Crowd Funding service may truncate this. No default.
- *for* URI relating pledge with some other item
- *redirect_uri* URI for redirecting client to afterwards
- *callback_uri* URI for performing a web callback

All of these are optional and user is just presented with a bare pledge form if left out.

#### Who is the recipient

'to' is interesting and presents us with a couple of semantic issues we need to think about.

When you first look at it you might think that to should be the recipient of the pledge (Alice). But really the beneficiary of all pledges within this project is Alice so the Supporter (Bob) is actually buying a "Pledge" asset to his own name. He could also pledge and pay on behalf of someone else, similar to the popular send money in your name to charity X christmas cards.

#### Examples

A perfectly valid Transfer Request would be sending the user to the OpenTransact url without query parameters.

    http://crowdstarter.example.com/projects/123123

You could think of a scenario where a local school asks people to support a project on their behalf.

    http://crowdstarter.example.com/projects/123123?to=fundraising@elmhurst-elementary.sample.edu

A Crowd funding aggregator might want to link people to projects and hear back from the crowd funding aggregator to create popularity statistics.

    http://crowdstarter.example.com/projects/123123?callback_uri=http://crowdedlist.sample.com/callback

Through these simple parameters many different mashups could be created to help people fundraise their projects.

#### Implementation

If none of the OpenTransact parameters are present the crowd funding service should just present their regular interface.

If any of the OpenTransact parameters are present the service fills out relevant parts of their interface and continues in [step 4](#pledge) above.

If a *redirect_uri* or *callback_uri* was present the Crowd funding service redirects the user back to the given redirect_uri and/or posts the receipt to the given callback_uri.

### Transfer

To allow transfers the Crowd Funding service needs to have an OAuth 2.0 implementation. There are implementations for just about any language now and this is in itself fairly straight forward.

Any application mobile or otherwise needs to register as a Client with the Crowd Funding service. They can then obtain and [authorize an OAuth Access token](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-4). This used to be quite complex for mobile apps, but the new [Password Credentials authorization](http://tools.ietf.org/html/draft-ietf-oauth-v2-22#section-4.3) is pretty straightforward from a usability standpoint.

In our case the user already has an account with the crowd funding service and has already linked their account with an OpenTransact payment provider using OAuth.

Lets assume the App already has a token "23d1760359159578aed0437e400f3b36" authorized by Bob.

- Bob browses the latest projects on his mobile crowd funding app and find Alice's project.
- He decides to support the project and pledge $20.
- The mobile app sends an OpenTransact Transfer of $20 to the OpenTransact url of the project
- CrowdStarter receives the request and verifies who is the user related to the OAuth token
- CrowdStarter reserves $20 at PayMe [Learn more about reserves](reserves.html)
- CrowdStarter creates a pledge in their database
- CrowdStarter returns a receipt to the mobile app


#### Implementation

The app sends the following to crowdstarter:

    POST /projects/123123 HTTP/1.1
    Host:  crowdstarter.example.com
    Authorization: Bearer 23d1760359159578aed0437e400f3b36
    Content-length: 9

    amount=20

CrowdStarter creates an initial Pledge in their system and sends the following to PayMe:

    POST /usd/reserves HTTP/1.1
    Host:  payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 105

    amount=20&for=https://crowdstarter.example.com/pledges/1231245&to=alice@project.example.com&validity=P34D

PayMe reserves $20 and returns the following:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "txn_url":"https://payme.example.com/reserves/1231245",
      "to": "alice@project.example.com",
      "from": "bob@project.example.com",
      "asset_url":"https://payme.example.com/usd",
      "amount":20,
      "expires_in":2937600,
      "for":"https://crowdstarter.example.com/pledges/1231245",
      "timestamp":"2012-01-11 22:08:48 UTC"
    }

CrowdStarter marks the pledge as valid and returns the following to the App:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "txn_url":"https://crowdstarter.example.com/pledges/1231245",
      "asset_url":"http://crowdstarter.example.com/projects/123123",
      "from": "alice@project.example.com",
      "to": "bob@project.example.com",
      "asset_url":"https://payme.example.com/usd",
      "amount":20,
      "expires_in":2937600,
      "for":"https://payme.example.com/transactions/1231245",
      "timestamp":"2012-01-11 22:08:48 UTC"
    }

That is the full flow and is a good example of a chained OpenTransact exchange transaction.

#### What happens if the user hasn't authorized payment?

This is a proposed pattern that we may bring into the standard.

If a transfer can not be done due to lack of balance. The HTTP standard defines [status code 402](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes#4xx_Client_Error) which could be used with a Location header field.

The user can then be redirected to that url to connect to a payment mechanism.

In practice CrowdStarter could return the OpenTransact url with the same OpenTransact parameters as in the Transfer

    http://crowdstarter.example.com/projects/123123?amount=20

They could return the following:

    HTTP/1.1 402 OK
    Location: http://crowdstarter.example.com/projects/123123?amount=20
    Cache-Control: no-store
    Pragma: no-cache

The mobile app can then open a web browser with the link and append a redirect_uri to their application:

    http://crowdstarter.example.com/projects/123123?amount=20&redirect_uri=com.sample.myapp:project

Once the user has authorized the payment in their mobile browser they get redirected back to the app.

