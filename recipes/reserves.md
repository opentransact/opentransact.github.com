---
title: Reserves
layout: default
---

# Reserves

A reserve is an amount of funds held in an account that expires at some stage. A credit card authorization is an example of a reserve.

The OpenTransact standard doesn't define reserves and how to use them. But these are best practices and may be written up into a separate OpenTransact Reserve standard later.

A reserve is not the same as a payment so it is a different asset as the main payments asset. We are currently using the term [derivative asset](http://www.opentransact.org/core.html#derivative-assets-1) for these assets that are derived from an existing asset yet have different semantics and business rules.

## Assets

The Payment service PayMe has as their primary OpenTransact URL:

    https://payme.example.com/usd

They could create a derivative opentransact url at:

    https://payme.example.com/usd/reserves

Transactions done there return a transaction url which in its own right an OpenTransact asset:

    https://payme.example.com/usd/reserves/123123

Until the reserve expires OpenTransact transfers can be created at this which creates real transactions in the primary asset.

## Applications

Reserves can be used as building blocks for exchanges and many other kind of applications including:

- [Crowd Funding](crowdfunding.html)
- Loan Guarantees
- Options

## Implementation example

We want to reserve $20 for a crowd funding application. In this application the client already has an OAuth token for the user:

### Creating reserve

We use the following OpenTransact parameters:

- amount=20 # amount to reserve
- for=https://crowdstarter.example.com/pledges/1231245 # A URI identifying the object being bought
- to=alice@project.example.com # The recipient of the reserve
- validity=P34D # An ISO8601 duration

All parameters are optional. But amount would normally be used.

Example defaults:

- amount defaults to the asset services specified default amount. Eg. $1
- to defaults to the OAuth clients account
- validity a default validity eg 1 week

The following is an example OpenTransact Transfer creating a reserve:

    POST /usd/reserves HTTP/1.1
    Host:  payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 105

    amount=20&for=https://crowdstarter.example.com/pledges/1231245&to=alice@project.example.com&validity=P34D

PayMe registers in their books that $20 is reserved and returns the following:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "txn_url":"https://payme.example.com/reserves/1231245",
      "to": "alice@project.example.com",
      "from": "bob@project.example.com",
      "asset_url":"https://payme.example.com/usd/reserves",
      "amount":20,
      "expires_in":2937600,
      "for":"https://crowdstarter.example.com/pledges/1231245",
      "timestamp":"2012-01-11 22:08:48 UTC"
    }

### Charging a reserve

We have a reserve for $20 with the following OpenTransact asset url:

    https://payme.example.com/reserves/1231245

#### Charge exactly as reserved

To charge the reserve without any changes just sent an empty http post to the reserves url:

    POST /usd/reserves/1231245 HTTP/1.1
    Host:  payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 0

This performs the actual payment in https://payme.example.com/usd of $20 to alice@project.example.com and returns the following:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "txn_url":"https://payme.example.com/payments/12314223",
      "to": "alice@project.example.com",
      "from": "bob@project.example.com",
      "asset_url":"https://payme.example.com/usd",
      "amount":20,
      "for":"https://crowdstarter.example.com/pledges/1231245",
      "timestamp":"2012-01-11 22:08:48 UTC"
    }

#### Charge a smaller amount

Maybe we want to charge various smaller amounts and not the full $20.

To charge the reserve while charging a smaller amount just sent incude the amount in a http post to the reserves url:

    POST /usd/reserves/1231245 HTTP/1.1
    Host:  payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 8

    amount=2

This performs the actual payment in https://payme.example.com/usd of $2 to alice@project.example.com and returns the following:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "txn_url":"https://payme.example.com/payments/12314223",
      "to": "alice@project.example.com",
      "from": "bob@project.example.com",
      "asset_url":"https://payme.example.com/usd",
      "amount":2,
      "for":"https://crowdstarter.example.com/pledges/1231245",
      "timestamp":"2012-01-11 22:08:48 UTC"
    }

#### Check balance of Reserve

We may want to check the amount of the reserve. This should show how much is left of the reserve in case it has already been partly charged.

The client issues this request:

    GET /usd/reserves/1231245 HTTP/1.1
    Host:  payme.example.com
    Accept: application/json
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA

The provider returns the receipt data showing the amount and can include a list of transactions created from it:

    HTTP/1.1 200 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

    {
      "txn_url":"https://payme.example.com/reserves/1231245",
      "to": "alice@project.example.com",
      "from": "bob@project.example.com",
      "asset_url":"https://payme.example.com/usd/reserves",
      "amount":18,
      "expires_in":2937600,
      "for":"https://crowdstarter.example.com/pledges/1231245",
      "timestamp":"2012-01-11 22:08:48 UTC",
      "transactions": [
        {
          "txn_url":"https://payme.example.com/payments/12314223",
          "to": "alice@project.example.com",
          "from": "bob@project.example.com",
          "asset_url":"https://payme.example.com/usd",
          "amount":2,
          "for":"https://crowdstarter.example.com/pledges/1231245",
          "timestamp":"2012-01-11 22:08:48 UTC"
        }
      ]
    }


#### Charge too large an amount

If someone tries to charge more than was reserved they are not allowed.

To charge the reserve while charging a smaller amount just sent incude the amount in a http post to the reserves url:

    POST /usd/reserves/1231245 HTTP/1.1
    Host:  payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 9

    amount=21

Since this is large than the reserved amount we return an HTTP 403 (Not allowed)

    HTTP/1.1 403 OK
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

#### Charge an expired reserve

If someone tries to charge a reserve after the reserve has expired we can use HTTP 410 (Gone) do indicate this.

    POST /usd/reserves/1231245 HTTP/1.1
    Host:  payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 0

We return 410

    HTTP/1.1 410 Gone
    Content-Type: application/json;charset=UTF-8
    Cache-Control: no-store
    Pragma: no-cache

#### Cancel a reserve

If for some reason we don't need a reserve anymore or we have charged part of it and don't need the rest, it can be cancelled.

    DELETE /usd/reserves/1231245 HTTP/1.1
    Host:  payme.example.com
    Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA
    Content-length: 0

We return the JSON with a field indicating it was cancelled:

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
      "for":"https://crowdstarter.example.com/pledges/1231245",
      "timestamp":"2012-01-11 22:08:48 UTC",
      "cancelled_at":"2012-01-13 22:08:48 UTC"
    }
