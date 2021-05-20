![Simple Ledger Protocol](images/SLP-logo-solid-200.png)

# Simple Ledger Protocol (SLP) Swap Protocol

#### Version: 0.1
#### Date published: May 20, 2021

## Purpose

This specification describes a protocol for peer-to-peer collaborative transactions on networks, such as Bitcoin Cash (BCH), that implement the Simple Ledger Protocol (SLP) for tokens. By using this protocol, users can effectuate instant, trustless, exchange of one SLP token for another.

## Background

Collaborative transactions are possible where two or more peers contribute unspent transaction outputs (UTXOs) containing varying amounts of both SLP and native BCH value and reassign this value to an output configuration of their choosing. This effectively creates the opportunity for the "exchange" of SLP tokens for BCH in a single transaction. As a concept, such transactions are the core of the [Simple Ledger Postage Protocol](https://github.com/simpleledger/slp-specifications/blob/master/slp-postage-protocol.md).

The Simple Ledger Protocol specification for Token Type 1, which is the current industry standard, does not allow for multiple different tokens (differing token ID's) to be used in a single transaction. This necessarily means that a collaborative transaction for the exchange of one token type for another must actually be done as a chain of transactions, broadcast simultaneously. The SLP Swap Protocol provides a standard for the creation of such a collaborative transaction chain.

## Specification

### Extending Existing Payment Protocols

This payment protocol is an extension of the [Simple Ledger Postage Protocol](https://github.com/simpleledger/slp-specifications/master/slp-postage-protocol.md) and the [JSON Merchant Data](https://github.com/jeton-tech/payment-protocol-extensions/blob/master/json-merchant-data.md) specifications. Applications supporting these protocols are able, with minor modifications, to support the protocol described in this specification.


### Exchange Rate Request

In order to determine the exchange rate for any two given tokens, the sender's wallet queries the SLP swap server. The server returns exchange rate data which can then be used by the wallet to calculate the appropriate amount of SLP token value to include in a payment output(s) as well as the proper payment address. Native BCH inputs are not required to send to the SLP swap server, since the server will provide the native BCH and subtract it from the token amount swapped back the original sender according to the [Simple Ledger Postage Protocol](https://github.com/simpleledger/slp-specifications/master/slp-postage-protocol.md).

#### REQUEST
A GET request should be made to the SLP Swap Protocol url. Standard endpoint is `https://someurl.com/rates`

#### Headers
* `Accept` should be set to `application/json`.

#### Response
The response will be a JSON format payload.

#### Response Body
* `version` - The version of the Postage Rate Request specification. This document defines version 1
* `address` - The address to which postage payment must be made
* `unit` - The native unit expressed by the rate (ie. BCH)
* `tokens` - An array of stamp objects, representing the tokens supported by the SLP swap server and the exchange rates, expressed in `unit` per one token
    * `name` - The human readable name of the SLP token
    * `symbol` - The ticker symbol of the token
    * `tokenId` - The token ID hash for the token
    * `decimals` - The number of decimal places for the token
    * `buy` - The rate at which the SLP swap server will receive the token
    * `sell` - The rate at which the SLP swap server will send the token
    * `available` - The number of tokens currently available for sale

#### Response Body Example
```
{
   "version":1,
   "address":"simpleledger:qpren52c6y98kt9eenjhtqe437dkwlyfcqj4j3dnmn",
   "unit":"BCH",
   "tokens":[
      {
         "name":"Spice Token",
         "symbol":"SPICE",
         "tokenId":"4de69e374a8ed21cbddd47f2338cc0f479dc58daa2bbe11cd604ca488eca0ddf",
         "decimals":8,
         "buy":"0.000001890",
         "sell":"0.000002100",
         "available":804.58609424
      },
      {
         "name":"Sai",
         "symbol":"SAI",
         "tokenId":"7853218e23fdabb103b4bccbe6e987da8974c7bc775b7e7e64722292ac53627f",
         "decimals":8,
         "buy":"0.0001834",
         "sell":"0.0002037",
         "available":0
      },
      {
         "name":"HONK HONK",
         "symbol":"HONK",
         "tokenId":"7f8889682d57369ed0e32336f8b7e0ffec625a35cca183f4e81fde4e71a538a1",
         "decimals":0,
         "buy":"0.000000001152",
         "sell":"0.000000001280",
         "available":1000000
      },
      {
         "name":"USDT",
         "symbol":"USDT",
         "tokenId":"9fc89d6b7d5be2eac0b3787c5b8236bca5de641b5bafafc8f450727b63615c11",
         "decimals":8,
         "buy":"0.000940",
         "sell":"0.001045",
         "available":2.1853512
      },
      {
         "name":"Honest Coin",
         "symbol":"USDH",
         "tokenId":"c4b0d62156b3fa5c8f3436079b5394f7edc1bef5dc1cd2f9d0c4d46f82cca479",
         "decimals":2,
         "buy":"0.000940",
         "sell":"0.001045",
         "available":0
      }
   ]
}
```

#### Integration With Simple Ledger Postage Protocol

A merchant utilizing [Simple Ledger Payment Protocol](https://github.com/simpleledger/slp-specifications/blob/master/slp-postage-protocol.md) to process payments can, optionally, act as an SLP Swap server. The merchant's server will add the required satoshis after validating the payment and before broadcasting the final chained transactions, effectuaing the swap, to the network. The buyer ("sender") specifies the token ID of the token to be received back (the result of the exchange) according to the [JSON Merchant Data](https://github.com/jeton-tech/payment-protocol-extensions/blob/master/json-merchant-data.md) specification.

#### Payment Request JSON Merchant Data Object Example (Buying [SPICE](https://explorer.bitcoin.com/bch/token/4de69e374a8ed21cbddd47f2338cc0f479dc58daa2bbe11cd604ca488eca0ddf))
```
{
   slpSwap: '4de69e374a8ed21cbddd47f2338cc0f479dc58daa2bbe11cd604ca488eca0ddf'
}
```


### Payment Generation

Payments are generated according to the Payment Generation section of the [Simple Ledger Payment Protocol](https://github.com/simpleledger/slp-specifications/blob/master/slp-payment-protocol.md) with the following additional considerations:

1. Both valid Simple Ledger Protocol and native (BCH) inputs and outputs may be included.
2. The sum of all valid SLP outputs directed to the `address` specified in the [Exchange Rate Request](#exchange-rate-request) response will constitute the value of the tokens being 'sold' to the SLP swap server.
3. At least one native (BCH) output must be directed to the `address` specified in the [Exchange Rate Request](#exchange-rate-request) response. This constitutes "dust" that will be used in the chained transaction which returns the "purchased" SLP token value to the original sender.
3. All inputs must use the SIGHASH_ALL | SIGHASH_ANYONECANPAY signature hash type. This is to mitigate any potential risk of malleation of outputs.
4. Sender broadcasts transaction to standard Postage endpoint of SLP Swap server, but uses the [MIME types specified in this document](#mime-types).
5. The SLP Swap server will broadcast the chained transactions to the network once the payment transaction has been validated and the corresponding swap transaction has been created using the dust, as specified in #3, as one of the inputs.

### Payment Validation

The SLP Swap server will validate any submitted payment, ensuring that it is an otherwise valid SLP transaction, before appending additional outputs (postage), creating the corresponding swap transaction, based on the current [exchange rates](#exchange-rate-request) (minus any cost for postage required) and broadcasting the transaction to the Bitcoin Cash network.

#### Example Transaction Chain: SPICE swapped for HONK using [example exchange rates][#response-body-example]

BEFORE SWAP:</br>

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | **(70 SPICE tokens)** 546 satoshis  | **SLP OP_RETURN** |
| 1 | | **(50 SPICE tokens)** 546 satoshis |
| 2 | | **(20 SPICE tokens)** 546 satoshis  *change* |
| 3 | | 546 satoshis  *dust (to chain)* |

AFTER SWAP:</br>

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | **(70 SPICE tokens)** 546 satoshis  | **SLP OP_RETURN** |
| 1 | 1148 satoshis *stamp* | To Swap Server **(50 SPICE tokens)** 546 satoshis |
| 2 | | To Sender **(20 SPICE tokens)** 546 satoshis  *change* |
| 3 | | To Swap Server 546 satoshis  *dust (to chain)* |

| INDEX | INPUT | OUTPUT |
| ------------ | ------------ | ------------------------------------------|
| 0 | **(193,828 HONK tokens)** 546 satoshis  | **SLP OP_RETURN** |
| 1 | 546 satoshis  *dust (to chain)* | To Sender **(73,828 HONK tokens minus stamp fee)** 546 satoshis |
| 2 | 1148 satoshis *stamp* | To Swap Server **(120,000 HONK tokens plus stamp fee)** 546 satoshis  *change* |


### MIME Types

The MIME (RFC 2046) Media Types follow the basic specification described in [Simple Ledger Payment Protocol](https://github.com/simpleledger/slp-specifications/blob/master/slp-payment-protocol.md) with the following modifications:

| Message  | Type/Subtype                                  |
| ------------ | ------------------------------------------|
| Payment      | application/simpleledger-swap          |
| PaymentACK      | application/simpleledger-paymentack    |

#### Example

A wallet client generating a Payment message would precede the binary message data with the following headers:

``Content-Type: application/simpleledger-swap``</br>
``Accept: application/simpleledger-paymentack``</br>
``Content-Transfer-Encoding: binary``</br>


### Common Errors

| HTTP Status Code | Response | Cause |
|---|---|---|
| 400 | Unsupported Content-Type for payment | Your Content-Type header was not valid |
| 400 | Invalid Payment | The payment is invalid in some way |
| 400 | Could not parse OP_RETURN output | SLP OP_RETURN output is malformed |
| 400 | Unsupported SLP token | The SLP token in the transaction is not among the supported tokens |
| 400 | Insufficient tokens paid | The amount paid is insufficient to cover postage and swap |
| 400 | Tokens currently unavailable | The swap server has insufficient token UTXOs available to sell. Please inform support |
| 500 | Error | An error has occurred on the swap server |


## Reference Implementations

### Clients
[slpswap-client](https://github.com/vinarmani/slpswap-client) - Node.js

### Libraries
None currently
