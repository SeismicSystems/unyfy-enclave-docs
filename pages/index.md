---
title: Elliptic Enclave WebSocket API Documentation
description: This document provides an overview and instructions on how to interact with the Elliptic Enclave WebSocket API.
---

## Overview

The Elliptic Enclave API is a WebSocket API that allows clients to interact with the enclave used for Elliptic's on-chain dark pool solution. This document provides detailed information on how to establish a connection, send messages, and place orders using this API.

## General Considerations

The API utilizes Amazon API Gateway to direct incoming requests to backend EC2 instances. These instances operate using Nitro enclaves and are orchestrated by Amazon EKS. 

**On 'subscription' and 'publication' (pub/sub)**: When the term "publication" is referenced within this documentation, it inherently means a private publication unless specifically mentioned otherwise. This ensures that users exclusively receive messages tailored to their individual private feed in the publication. Naturally, then, it follows that "subscription" refers to subscribing to these private feeds.

### Establishing a Connection

To initiate a connection to the API, employ the command below. Once connected, API Gateway will trigger a route that captures and records your unique connection ID.

```javascript
const socket = new WebSocket('wss://abcdef123.execute-api.us-west-2.amazonaws.com/production');

socket.addEventListener('open', function (event) {
    console.log('Connected');
    // Prompt the server for the challenge message
    socket.send(JSON.stringify({ action: "request_challenge" }));
});

socket.addEventListener('message', function (event) {
    console.log('Message from server ', event.data);
    const data = JSON.parse(event.data);

    if (data.action === "challenge") {
        const challenge = data.challenge;
        // Subsequently, sign the challenge and transmit the authentication payload.
    }
});
```

### Authentication with Ethereum Public Key

Upon connection, a challenge message will be dispatched to you. This challenge necessitates being signed with your Ethereum private key, confirming the ownership of the associated Ethereum address.

On obtaining the challenge:

1. Sign it employing your Ethereum wallet.
2. Return the signed challenge along with your Ethereum public key to the server via the WebSocket connection.

```javascript
const web3 = new Web3(/* Your web3 provider here */);
const challenge = "ReceivedChallengeFromServer";  // Swap with the actual received challenge
const ethereum_public_key = "YOUR_ETHEREUM_PUBLIC_KEY";  // Amend with your Ethereum public key

web3.eth.personal.sign(challenge, ethereum_public_key, null, (error, signature) => {
    if (!error) {
        const authPayload = {
            action: "authenticate",
            publicKey: ethereum_public_key,
            signature: signature
        };

        socket.send(JSON.stringify(authPayload));
    } else {
        console.error("Error signing the challenge:", error);
        // If there's an error in the signing process, close the connection
        socket.close();
    }
});
```

### Connection Timeout

It's imperative to respond to the challenge within a stipulated time frame to ensure prompt authentication. A 5-minute window is provided post the challenge issuance. If the authentication payload (signed message and public key) isn't received within this span, the server will terminate the connection.


### Subscription
**Request.** To subscribe to a feed, you need to send a message to the server after establishing a connection. The message should be a JSON object with an `action` property set to `subscribe`, a `feed` property set to the name of the feed you want to subscribe to. Note that no additional information is required to be submitted for subscribing to a publication since the WebSocket connection only persists once a challenge message is signed with the user's Ethereum public key, hence not necessiating the need for another authentication mechanism.

Here is an example of how to subscribe to the 'openorders' feed using JavaScript:

```javascript
const subscription = {
  action: 'subscribe',
  feed: 'openorders'
};

socket.send(JSON.stringify(subscription));
```

In this example, we are subscribing to the 'openorders' feed. The server will start sending messages for the 'openorders' feed as soon as the subscription is successful.

The payload for the 'openorders' feed subscription is as follows:

**Payload:**
 Field   | Type    | Description                                                                                     
---------|---------|-------------------------------------------------------------------------------------------------
 `action`| `string`| The action being performed, "subscribe".                                                       
 `feed`  | `string` | The name of the feed to subscribe to, in this case 'openorders'.                              
                             


### Orders

#### sendorder

**Request.** Client can send a raw order (```O```) to the server, server responds with an enclave signature (σ).

**Payload:**
 Field                         |  Type     | Description                                                                                     
-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
 `action`                      | `string` | The action to be performed, "sendorder".                                                        
 `data`                        | `object` | The order data.                                                                                 
 &nbsp;&nbsp;`transparent`     | `object` | The transparent structure of the order.                                                         
 &nbsp;&nbsp;&nbsp;&nbsp;`side`| `number` | Side of the order, 0 for bid, 1 for ask.                                                        
 &nbsp;&nbsp;&nbsp;&nbsp;`token` | `string` | Token address for the target project.                                                           
 &nbsp;&nbsp;&nbsp;&nbsp;`denomination` | `string` | Either the token address or USDC or ETH (set to 0x1 for this case).              
 &nbsp;&nbsp;`shielded`         | `object` | The shielded structure of the order.                                                            
 &nbsp;&nbsp;&nbsp;&nbsp;`price` | `number` | Price, in `denomination`. Scaled by 10^9 but only has 10^7 precision.                           
 &nbsp;&nbsp;&nbsp;&nbsp;`volume` | `number` | Volume: amount of token to exchange. Scaled by 10^9.                                            
 &nbsp;&nbsp;&nbsp;&nbsp;`accessKey` | `string` | Access key: protects from brute force attacks, revealed to counterparties
 `hash`                        | `string` | Hash of the shielded portion of the order, which forms the shielded portion of the order commitment.


Example of payload:

```javascript
{
  "action": "sendorder",
  "data": {
    "transparent": {
      "side": 0,
      "token": "token_address",
      "denomination": "0x1"
    },
    "shielded": {
      "price": 1000000000,
      "volume": 1000000000,
      "accessKey": "access_key"
    }
  }
  "hash": "hash_of_shielded_portion_of_order"
}


```

#### getenclavesignature

**Response.** Server's response after receiving a client's order through through ```sendorder```. The server returns an enclave signature. 

**Payload:**
 Field                         |  Type     | Description                                                                                     
-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
 `action`                      | `string` | The action to be performed, "getenclavesignature".                                                                                                                   
 `enclaveSignature`            | `string` | The enclave signature of the order.                                  
 
 `orderCommitment`             | `object` | The orderCommitment that the enclave signs.                                                                                                                    
 &nbsp;&nbsp;`transparent`     | `object` | The transparent part of the order, includes side, token, denomination.                                                        
 &nbsp;&nbsp;&nbsp;&nbsp;`side`| `number` | Side of the order, 0 for bid, 1 for ask.                                                        
 &nbsp;&nbsp;&nbsp;&nbsp;`token` | `string` | Token address for the target project.                                                           
 &nbsp;&nbsp;&nbsp;&nbsp;`denomination` | `string` | Either the token address or USDC or ETH (set to 0x1 for this case).              
 &nbsp;&nbsp;`shielded`        | `string` | The shielded part of the order commitment (hash of shielded part of the raw order.)

Example of payload:

```javascript
{
  "action": "getenclavesignature",
    "enclaveSignature": "enclave_signature_value",
    "orderCommitment": {
      "transparent": {
        "side": 0,
        "token": "token_address",
        "denomination": "0x1"
      },
      "shielded": "hash_of_shielded_part_of_order"
    }
  }
```

#### getcrossedorders

**Publication.** The enclave publishes the crossed orders. We use the hash of the shielded part of the order (```H(O.s)```) as a unique identifier for the order.

**Payload:**
| Field                | Type    | Description                                                                                     |
|----------------------|---------|-------------------------------------------------------------------------------------------------|
| `action`             | `string`| The action being performed, "getcrossedorders".                                                |
| `hash` | `string`| The unique identifier of the order for which these orders are crossed/matched.                  |
| `orders`             | `array` | An array of crossed order objects. Each object contains details of the order.                   |

Example of payload:

```javascript
{
  "action": "getcrossedorders",
  "hash": "hashxyz",
  "orders": [
    {
      "orderId": "crossedOrder123",
      "orderCommitment": "commitment_value_1",
      "timestamp": 1678901234567,
      "transparent": {
        "side": 0,
        "token": "token_address_1",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 1000000000,
        "accessKey": "access_key_1"
      }
    },
    {
      "orderId": "crossedOrder124",
      "orderCommitment": "commitment_value_2",
      "timestamp": 1678901234578,
      "transparent": {
        "side": 1,
        "token": "token_address_2",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1100000000,
        "volume": 900000000,
        "accessKey": "access_key_2"
      }
    }
  ]
}

```
#### orderhistory

**Publication.** The enclave publishes the order history for a client.

**Payload:**
| Field   | Type    | Description                                                                                     |
|---------|---------|-------------------------------------------------------------------------------------------------|
| `action`| `string`| The action being performed, "orderhistory".                                                     |
| `orders`| `array` | An array of historical order objects. Each object contains details of the order.                |

Example of payload:
```javascript
{
  "action": "orderhistory",
  "orders": [
    {
      "orderId": "order12345",
      "timestamp": 1678901234567,
      "status": "matched",
      "transparent": {
        "side": 0,
        "token": "token_address_1",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 1000000000,
        "accessKey": "access_key_1"
      }
    },
    {
      "orderId": "order12346",
      "timestamp": 1678901234578,
      "status": "unmatched",
      "transparent": {
        "side": 1,
        "token": "token_address_2",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1100000000,
        "volume": 900000000,
        "accessKey": "access_key_2"
      }
    }
  ]
}
```
#### openorders

**Publication.** The enclave publishes the open orders for a client.

**Payload:**
| Field   | Type    | Description                                                                                     |
|---------|---------|-------------------------------------------------------------------------------------------------|
| `action`| `string`| The action being performed, "openorders".                                                       |
| `orders`| `array` | An array of open order objects. Each object contains details of the order.                      |

Example of payload:
```javascript
{
  "action": "openorders",
  "orders": [
    {
      "orderId": "order12345",
      "timestamp": 1678901234567,
      "transparent": {
        "side": 0,
        "token": "token_address_1",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 1000000000,
        "accessKey": "access_key_1"
      }
    },
    {
      "orderId": "order12346",
      "timestamp": 1678901234578,
      "transparent": {
        "side": 1,
        "token": "token_address_2",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1100000000,
        "volume": 900000000,
        "accessKey": "access_key_2"
      }
    }
  ]
}
```