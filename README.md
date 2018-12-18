# Veil API (beta)

The Veil REST API provides endpoints to interact with Veil in all the ways possible through our [web app](https://kovan.veil.market). You can list and filter markets, get open orders and trade histories, and create and cancel your own orders in open markets.

## A word of caution

Veil is still in the early stages of development, and we expect the API to change over time. While the API remains in beta, backwards compatibility is best-effort. During that time, we're happy to work with you directly on keeping your integrations up-to-date.

## Questions?

Join us on [Discord](https://discord.gg/aBfTCVU) or email us at `hello@veil.market`. If you have a recommendation about the API documentation, feel free to [open an issue](https://github.com/veilco/veil-api-docs/issues).

The Veil API is maintained by @gkaemmer, @mertcelebi, and @pfletcherhill.

## `veil-js`

If you are using JavaScript or TypeScript, we recommend checking out [`veil-js`](https://github.com/veilco/veil-js). It's a thin wrapper around the API and automatically handles things such as authentication and signing orders.

## Contents

- [Sending requests](#sending-requests)
- [Units and data types](#units-and-data-types)
- [Shares primer](#shares-primer)
- [Authentication](#authentication)
- Public endpoints
    + [`GET /markets`](#get-markets)
    + [`GET /markets/:slug`](#get-marketsslug)
    + [`GET /markets/:slug/(long|short)/bids`](#get-marketssluglongshortbids-and-get-marketssluglongshortasks)
    + [`GET /markets/:slug/(long|short)/asks`](#get-marketssluglongshortbids-and-get-marketssluglongshortasks)
    + [`GET /markets/:slug/(long|short)/order_fills`](#get-marketssluglongshortorder_fills)
- Authentication endpoints
    + [`POST /session_challenges`](#post-session_challenges)
    + [`POST /sessions`](#post-sessions)
- Authenticated endpoints
    + [`GET /orders`](#get-orders)
    + [`POST /quotes`](#post-quotes)
    + [`POST /orders`](#post-orders)
    + [`DELETE /orders/:uid`](#delete-ordersuid)

### Sending requests

The API prefix is `https://api.kovan.veil.market/api/v1`. For example, to get all markets, you'd send a request to `https://api.kovan.veil.market/api/v1/markets`.

All successful requests will contain a `data` key at the root, whereas failing requests will return an `error` key:
```json
{
  "data": { "results": { ... }, ... }
}
// OR
{
  "errors": [{ "message": "Market not found", "code": "OTHER_ERROR" }]
}
```

### Units and data types

- Dates are returned as an integer number of milliseconds since the Unix epoch.
- Numbers are returned as strings for precision.
  + The length of a number depends on its type. Consider the following order:
  ```json
  {
    "currency_amount": "229300000000000000",
    "price": "2293",
    "token_amount": "100000000000000",
    ...
  }
  ```
    + `currency_amount` is denominated in wei (`10e-18 ETH`). Dividing by `10e18` gives us an ETH amount, `0.2293 ETH`.
    + `price` is always an integer between 0 and 10000. Here, the price is `0.2293 ETH/SHARE`
    + `token_amount` can be divided by `10e14` to give the number of shares, which in this case is exactly `1.00 SHARE`.
    + Therefore, `currency_amount` (in ETH) is always equal to `price` multiplied by `token_amount`.
- Lists are returned as pages, which have the following form:
  ```json
  {
    "results": [ ... ],
    "total": 10,
    "page": 0,
    "pageSize": 100
  }
  ```

## Shares primer

Veil markets are built on [Augur](https://docs.augur.net/) and inherit the basic mechanics of Augur shares and prices.

A Veil market has two tokens: LONG and SHORT. By holding shares of LONG or SHORT tokens, you hold a "position" in the market. When the market ends, its LONG and SHORT shares are redeemable for ETH, with the rates depending on the market's result.

In yes/no markets (e.g. "Will ETH be above $100 at the end of 2018?"), the payout goes entirely to one share token—LONG if the market resolves to "Yes" and SHORT if the market resolves to "No".

In scalar markets (e.g. "What will be the price of ETH at the end of 2018?"), the payout is split between LONG and SHORT tokens according to where the result (e.g. the price of ETH) falls within the market's "bounds" (set by `min_price` and `max_price`).

> Note: Together, 1 LONG share and 1 SHORT share are always redeemable for exactly 1 ETH.

The price of a Veil share token is therefore always somewhere between 0 and 1 ETH per share, depending on what the market predicts that each token's payout will be.

Share token prices are expressed as integers between 0 and 10000, where 10000 means 1 ETH/share. A token with a price of 6000 can be purchased for 0.6 ETH/share.

### Authentication

Veil uses authentication tokens to restrict access to private methods (quote and order creation, order cancelation, etc.). The token is passed using the `Authorization` header as follows:
```
Authorization: Bearer [TOKEN]
```

To get a temporary authentication token, you must sign a unique, one-time "session challenge" using an ethereum address. The ethereum address you use must be [registered on Veil](https://kovan.veil.market). Session challenges can be created using the `POST /session_challenges` endpoint.

To sign a session challenge, you can sign any UTF-8 string containing its `uid`. Signing is performed using the ethereum `eth_sign` standard.

To create a session, you pass the signature to the `POST /sessions` endpoint, which returns the `token`.

For an example implementation of authentication, take a look at the [`authenticate` method of `veil-js`](https://github.com/veilco/veil-js/blob/master/src/Veil.ts).

### Public endpoints

### `GET /markets`

Fetch a list of markets.

#### Params
- `status`: `open` or `resolved`. Filters markets by their status. If omitted, both open and resolved markets are returned.
- `channel`: `btc`, `rep`, `zrx`, `gas`, `hash_rate`, or `memes`. Filters markets by the channel they're in.
- `page`: An 0-based integer page number to request. A maximum of 10 markets are returned per page.

#### Example
`GET /markets?status=open&channel=zrx`:
```json
{  
  "data": {  
    "results": [  
      {  
        "name": "What will be the price of ZRX in USD at 12am UTC on December 22, 2018?",
        "address": "0xcd301ba9478e42706e493d7a5c822f6e2d6c2111",
        "details": "Will resolve at the last trade price before midnight UTC on CryptoCompare.",
        "created_at": 1544832033056,
        "ends_at": 1545436800000,
        "num_ticks": "10000",
        "min_price": "160000000000000000",
        "max_price": "380000000000000000",
        "limit_price": null,
        "type": "scalar",
        "uid": "acbe139b-c423-4746-b02e-d2bc6279020e",
        "slug": "zrx-usd-2018-12-22",
        "result": null,
        "long_buyback_order": null,
        "short_buyback_order": null,
        "long_token": "0x87f73640e6a260f1e68db783b8b6824496b2c8c8",
        "short_token": "0x31fc9c86da18a581db52a7b1fad291d9b39ac024",
        "denomination": "USD",
        "channel": "zrx",
        "index": "zrx-usd",
        "predicted_price": "6020",
        "metadata": {},
        "final_value": null
      }
    ],
    "total": 1,
    "page": 0,
    "page_size": 10
  }
}
```

### `GET /markets/:slug`

Fetch a single market using its "slug" (a relatively short human-readable identifier).

#### Params
*None*

#### Example
`GET /markets/zrx-usd-2018-12-22`:
```json
{
  "data": {
    "name": "What will be the price of ZRX in USD at 12am UTC on December 22, 2018?",
    "address": "0xcd301ba9478e42706e493d7a5c822f6e2d6c2111",
    "details": "Will resolve at the last trade price before midnight UTC on CryptoCompare.",
    "created_at": 1544832033056,
    "ends_at": 1545436800000,
    "num_ticks": "10000",
    "min_price": "160000000000000000",
    "max_price": "380000000000000000",
    "limit_price": null,
    "type": "scalar",
    "uid": "acbe139b-c423-4746-b02e-d2bc6279020e",
    "slug": "zrx-usd-2018-12-22",
    "result": null,
    "long_buyback_order": null,
    "short_buyback_order": null,
    "long_token": "0x87f73640e6a260f1e68db783b8b6824496b2c8c8",
    "short_token": "0x31fc9c86da18a581db52a7b1fad291d9b39ac024",
    "denomination": "USD",
    "channel": "zrx",
    "index": "zrx-usd",
    "predicted_price": "6020",
    "metadata": {},
    "final_value": null
  }
}
```

### `GET /markets/:slug/(long|short)/bids` and `GET /markets/:slug/(long|short)/asks`

Fetch the bids (open buy orders) or asks (open sell orders) for a market. You can fetch orders for either LONG or SHORT tokens (in Veil markets, the LONG and SHORT order books are always mirror images of each other).

Bids are sorted by price descending, and asks are sorted by price ascending, so you can get the spread of a market by comparing the first bid and first ask.

#### Params
*None*

#### Example
`GET /markets/zrx-usd-2018-12-22/long/bids`
```json
{  
  "data": {  
    "results": [  
      {  
        "token_amount": "100000000000000",
        "price": "5620"
      }
    ],
    "total": 1,
    "page": 0,
    "page_size": 10000
  }
}
```

### `GET /markets/:slug/(long|short)/order_fills`

Fetch the order fill history in a market for either LONG or SHORT tokens. 

#### Params
*None*

#### Example
`GET /markets/zrx-usd-1-2019-01-01/long/order_fills`
```json
{
  "data": {
    "results": [
      {
        "uid": "37bd71f2-8947-4451-99ba-038690bcbba5",
        "token_amount": "10000000000000",
        "price": "350",
        "status": "completed",
        "created_at": 1545036189196,
        "side": "sell"
      },
      {
        "uid": "fafb2f7b-9bc5-42ad-83be-018a0ffdfe48",
        "token_amount": "10362694300518",
        "price": "350",
        "status": "completed",
        "created_at": 1543277830930,
        "side": "sell"
      },
      ...
    ],
    "total": 28,
    "page": 0,
    "page_size": 100
  }
}
```

### `POST /session_challenges`

Create a session challenge. The `uid` of the session challenge must be included in a signed message passed to `POST /sessions`.

For more information, see [Authentication](#authentication).

#### Params
*None*

#### Example
`POST /session_challenges`:
```json
{
  "data": {
    "created_at": 1545089935438,
    "uid": "33d2d822-7956-4ab3-9dfd-a5e9d64c886b"
  }
}
```

### `POST /sessions`

Create a session by passing a signed session challenge. The `uid` of the session challenge must be included in a signed message passed to `POST /sessions`.

You must sign the challenge with an address that is already registered on Veil.

For more information, see [Authentication](#authentication).

#### Params
- `challenge_uid`: The `uid` of the session challenge used to create this session.
- `message`: Any message that contains the challenge's `uid`. For simplicity, this can just be the exact `uid`, unchanged. (This is used to give users more context when signing online.)
- `signature`: The result of signing `message`. To obtain this value, call `eth_sign` with `message`. The signature should be in hexadecimal and prefixed with `0x` (the standard format is RSV, so make sure that your signature library is consistent).

#### Example
```
POST /sessions
{
  "challenge_uid": "717ababf-9f71-4c3c-b43f-f007ac0a3fb4",
  "message": "717ababf-9f71-4c3c-b43f-f007ac0a3fb4",
  "signature": "0x4c19c7a293e5e85360c95fd539329cc654747cd781c2f17718ae67be3dfaf12c19070517a503d1f5797480f885a8867603cc7003d0ec6657b7cc6a511a5a64981c"
}
```
:
```json
{
  "data": {
    "token": "eyJhbG[...]",
    "account": {
      "uid": "549a30ea-2e36-462c-ae44-d0202b803547",
      "created_at": 1545036189196
      "email": "example@blah.com"
      "address": "0x1234567890abcdef1234567890abcdef12345678"
      "username": "example"
    }
  }
}
```

### `GET /orders`

Requires authentication. Fetches all orders that you've created in a particular market, including orders that have been filled.

#### Params
- `market` (**required**): The slug of a market.

#### Example
`GET /orders`:
```json
{
  "data": {
    "results": [
      {
        "uid": "3e4fd40d-176f-432f-8f0b-d0a600d55a1f",
        "status": "filled",
        "created_at": 1543509213537,
        "expires_at": 100000000000000,
        "type": "limit",
        "token_type": "short",
        "side": "buy",
        "price": "2293",
        "token": "0x598b46d68e3e03f810a45d7d8dc9af5afdbafd56",
        "token_amount": "100000000000000",
        "token_amount_filled": "0",
        "currency": "0xe7a67a41b4d41b60e0efb60363df163e3cb6278f",
        "currency_amount": "229300000000000000",
        "currency_amount_filled": "0",
        "post_only": false,
        "market": null
      },
      ...
    ],
    "total": 10,
    "page": 0,
    "page_size": 10000
  }
}
```

### `POST /quotes`

Requires authentication. Create a Veil quote, which is used to calculate fees and generate an unsigned 0x order, which is in turn required to create a Veil order.

#### Params
- `quote` (**required**): An object with the following fields:
  + `side` (**required**): Either `"buy"` or `"sell"`
  + `token` (**required**): The address of the token you wish to buy. Can be obtained from `market.long_token` or `market.short_token`.
  + `token_amount` (**required**): The amount of `token` you wish to buy or sell. Note that this number is expressed as an integer shifted by 14 decimal places. To purchase "1 share" in a Veil market, you would set `token_amount` to `"100000000000000"`
  + `price` (**required**): The price you wish to buy or sell at, expressed as an stringified integer greater than 0 and less than 10000.
  + `type` (**required**): This must be `"limit"`. Support for market orders is on its way.

#### Example
```
POST /quotes
{
  "quote": {
    "side": "buy",
    "token": "0x598b46d68e3e03f810a45d7d8dc9af5afdbafd56",
    "token_amount": "100000000000000",
    "price": "4000",
    "type": "limit"
  }
}
```
Example response:
```json
{
  "uid": "5d93b874-bde1-4af1-b7af-ae726943f549",
  "order_hash": "0x39c5934cff5e608743f845a8c6950cc897ed75d8127023887d9715fa3c60c27c",
  "created_at": 1543510274469,
  "expires_at": 100000000000000,
  "quote_expires_at": 1543510334469,
  "token": "0x598b46d68e3e03f810a45d7d8dc9af5afdbafd56",
  "currency": "0xe7a67a41b4d41b60e0efb60363df163e3cb6278f",
  "side": "buy",
  "type": "limit",
  "currency_amount": "400000000000000000",
  "token_amount": "100000000000000",
  "fillable_token_amount": "0",
  "fee_amount": "4000000000000000",
  "price": "4000",
  "zero_ex_order": {
    "salt": "35666599517228498817069108086005958238926633694259560734477953229163342485507",
    "maker_fee": "0",
    "taker_fee": "0",
    "maker_address": "0x1234567890abcdef1234567890abcdef12345678",
    "taker_address": "0xe779275c0e3006fe67e9163e991f1305f1b6fe99",
    "sender_address": "0xe779275c0e3006fe67e9163e991f1305f1b6fe99",
    "maker_asset_data": "0xf47261b0000000000000000000000000e7a67a41b4d41b60e0efb60363df163e3cb6278f",
    "taker_asset_data": "0xf47261b0000000000000000000000000598b46d68e3e03f810a45d7d8dc9af5afdbafd56",
    "exchange_address": "0x35dd2932454449b14cee11a94d3674a936d5d7b2",
    "maker_asset_amount": "404000000000000000",
    "taker_asset_amount": "100000000000000",
    "fee_recipient_address": "0x0000000000000000000000000000000000000000",
    "expiration_time_seconds": "100000000600"
  }
}
```

### `POST /orders`

Requires authentication. Creates an order using a generated quote. To call this method, you must first create a quote and sign the quote's `zero_ex_order`, which you then pass as an argument to this endpoint.

#### Params
- `order` (**required**): An object with the following fields:
  + `zero_ex_order` (**required**): The **signed** 0x order.
  + `quote_uid` (**required**): The `uid` of the quote that was used to create this order.

#### Example
```
POST /orders
{
  "orders": {
    "quote_uid": "5d93b874-bde1-4af1-b7af-ae726943f549",
    "zero_ex_order": {
      "salt": "35666599517228498817069108086005958238926633694259560734477953229163342485507",
      "maker_fee": "0",
      "taker_fee": "0",
      "maker_address": "0x1234567890abcdef1234567890abcdef12345678",
      "taker_address": "0xe779275c0e3006fe67e9163e991f1305f1b6fe99",
      "sender_address": "0xe779275c0e3006fe67e9163e991f1305f1b6fe99",
      "maker_asset_data": "0xf47261b0000000000000000000000000e7a67a41b4d41b60e0efb60363df163e3cb6278f",
      "taker_asset_data": "0xf47261b0000000000000000000000000598b46d68e3e03f810a45d7d8dc9af5afdbafd56",
      "exchange_address": "0x35dd2932454449b14cee11a94d3674a936d5d7b2",
      "maker_asset_amount": "404000000000000000",
      "taker_asset_amount": "100000000000000",
      "fee_recipient_address": "0x0000000000000000000000000000000000000000",
      "expiration_time_seconds": "100000000600",
      "signature": "0x1bd1488919bd9c1d3dacb5b5b13fc1774f95d8279946ce75b719bfb319136645337015f77e38ca8afa651603dbdff5df9d94c7d182cd84f9ea4cf632531226c69b03"
    }
  }
}
```
Example response:
```json
{
  "uid": "77fb963c-6b78-48ce-b030-7a08246e1f9f",
  "status": "open",
  "created_at": 1543510274884,
  "expires_at": 100000000000000,
  "type": "limit",
  "token_type": "short",
  "side": "buy",
  "price": "4000",
  "token": "0x598b46d68e3e03f810a45d7d8dc9af5afdbafd56",
  "token_amount": "100000000000000",
  "token_amount_filled": "0",
  "currency": "0xe7a67a41b4d41b60e0efb60363df163e3cb6278f",
  "currency_amount": "400000000000000000",
  "currency_amount_filled": "0",
  "post_only": false,
  "market": null
}
```

### `DELETE /orders/:uid`

Requires authentication. Cancels an order.

#### Params
*None*

#### Example
`DELETE /orders/77fb963c-6b78-48ce-b030-7a08246e1f9f`:
```json
{
  "uid": "77fb963c-6b78-48ce-b030-7a08246e1f9f",
  "status": "open",
  "created_at": 1543510274884,
  "expires_at": 100000000000000,
  "type": "limit",
  "token_type": "short",
  "side": "buy",
  "price": "4000",
  "token": "0x598b46d68e3e03f810a45d7d8dc9af5afdbafd56",
  "token_amount": "100000000000000",
  "token_amount_filled": "0",
  "currency": "0xe7a67a41b4d41b60e0efb60363df163e3cb6278f",
  "currency_amount": "400000000000000000",
  "currency_amount_filled": "0",
  "post_only": false,
  "market": null
}
```
