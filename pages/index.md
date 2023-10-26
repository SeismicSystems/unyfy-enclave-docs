---
title: Elliptic Enclave WebSocket API Documentation
description: This document provides an overview and instructions on how to interact with the Elliptic Enclave WebSocket API.
---

## Overview

Unyfy's clients can communicate with Elliptic via a WebSocket API. This document provides detailed information on how to establish a connection, send requests, and subscribe to feeds using this API.

## Connecting and Authenticating a Client

The API utilizes Amazon API Gateway to direct incoming requests to Nitro enclaves.

### Acquiring JWT

Before establishing a WebSocket connection, a JWT (JSON Web Token) needs to be acquired by verifying the ownership of an Ethereum public key with funds deposited in the contract. The following steps outline the process of obtaining a JWT:

1. **Request Challenge**:
  - The client sends a request to the issuing server to initiate the challenge-response mechanism.
  - The server generates a unique challenge.

```javascript
const axios = require('axios');

async function requestChallenge(ethPublicKey) {
    const response = await axios.post(env['ISSUING_SERVER_ENDPOINT'], {
        ethPublicKey: ethPublicKey
    });
    return response.data.challenge;
}

const ethPublicKey = env['YOUR_ETH_PUBKEY'];
requestChallenge(ethPublicKey).then(challenge => {
    console.log('Received challenge:', challenge);
});
```

2. **Sign Challenge**:
  - The client signs the challenge using their Ethereum private key.

```javascript 
const Web3 = require('web3');
const web3 = new Web3(/* Your web3 provider here */);

async function signChallenge(challenge) {
    const account = web3.eth.accounts.privateKeyToAccount(env['YOUR_ETH_PRIVKEY']);
    const signature = await account.sign(challenge);
    return signature.signature;
}
```

3. **Verify Challenge and Obtain JWT**:
  - The client sends the signed challenge back to the server.
  - The server verifies 1) the signature corresponds to the claimed public key and 2) the public key has funds deposited in the contract.
  - If the verification is successful, the server issues a JWT to the client.
```javascript
async function getJWT(signedChallenge) {
    const response = await axios.post('https://jwt-issuer.example.com/verify-challenge', {
        signedChallenge: signedChallenge
    });
    return response.data.token;
}

signChallenge(challenge).then(signedChallenge => {
    getJWT(signedChallenge).then(token => {
        console.log('Received JWT:', token);
    });
});
```
### JWT Expiration

The issued JWT has a validity period of 15 minutes from the time of issuance. This means that the token must be used for authentication within this time frame, else it will expire and a new JWT will need to be obtained through the aforementioned process. The expiration time is a security measure to ensure that tokens are used promptly and to minimize the risk associated with compromised or leaked tokens.

The expiration time of a JWT is typically represented by the `exp` claim in the payload section of the token, which is a timestamp indicating the expiration time. The server will check this claim to determine whether the token is still valid.

It's advisable to handle token expiration gracefully in your client application. For instance, you may want to implement logic to request a new token and re-establish the connection if the token expires while the application is running.

### Establishing a connection

To initiate a connection to the Websocket API, employ the command below. Upon connection, API Gateway will trigger a route that captures and records your unique connection ID. The WebSocket connection requires a valid JWT for authentication.

```javascript
const socket = new WebSocket('wss://abcdef123.execute-api.us-west-2.amazonaws.com/production');

socket.addEventListener('open', function (event) {
    console.log('Connected');
    // Authenticate the connection using the JWT
    socket.send(JSON.stringify({ action: "authenticate", token: /* yourJWTTokenHere */ }));
});

socket.addEventListener('message', function (event) {
    console.log('Message from server ', event.data);
    // Handle server messages here
});
```
In the code snippet above, replace ```yourJWTTokenHere``` with the JWT token obtained from the REST API in the previous steps.


## Submitting Requests to the Orderbook

### Getting Open Orders
A client can request all open orders associated with their Ethereum public key:

**Request Payload:**
 Field                         |  Type     | Description                                                                                     
-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
 `action`                      | `string` | The action to be performed, "openorders". 

**Response Payload:**
| Field   | Type    | Description                                                                                     |
|---------|---------|-------------------------------------------------------------------------------------------------|
| `action`| `string`| The action being performed, "openorders".                                                       |
| `orders`| `array` | An array of open order objects. Each object contains details of the order.      

**Order Object:**

Each order object in the `orders` array has the following structure:

| Field                         |  Type     | Description                                                                                     
|-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
| `orderCommitment`             | `object` | Contains the order commitment details.                                                                                 
| &nbsp;&nbsp;`transparent`     | `object` | Contains transparent details of the order.                                                         
| &nbsp;&nbsp;&nbsp;&nbsp;`side`| `number` | Side of the order, 0 for bid, 1 for ask.                                                        
| &nbsp;&nbsp;&nbsp;&nbsp;`token` | `string` | Token address for the target project.                                                           
| &nbsp;&nbsp;&nbsp;&nbsp;`denomination` | `string` | Either the token address or USDC or ETH (set to 0x1 for this case).              
| &nbsp;&nbsp;`shielded`         | `string` | The shielded structure of the order commitment, i.e. the hash of the shielded portion of the raw order.

**Example Request Payload:**
```javascript
const req = {
  action: 'openorders'
};

socket.send(JSON.stringify(req));
```

**Example Response Payload:**
```javascript
{
  "action": "openorders",
  "orders": [
    {
    "orderCommitment": {
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x4e3c3e3e4c3d4b4c5b5a5b5c6d6e6f7e8a8b8c9d1a1b1c2d3e4f5g6h7i8j9k0l1m2"
    },
      "timestamp": 1678901234567,
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 500000000,
        "accessKey": "0x123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0" 
      }
    },
    {
      "orderCommitment": {
      "transparent": {
        "side": 0,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c"
    },
      "timestamp": 1678901234578,
      "transparent": {
        "side": 0,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 500000000,
        "accessKey": "0xe123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0"
      }
    }
  ]
}
```

### Placing an Order
A client can send raw order (```O```) to the server, server responds with an enclave signature (σ) of the corresponding commitment (```Ō```).

**Request Payload:**
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
 `hash`                        | `string` | Hash of the shielded portion of the order, which forms the shielded portion of the order commitment, i.e. hash of the `shielded` object above.

**Response Payload:**
 Field                         |  Type     | Description                                                                                     
-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
 `action`                      | `string` | The action to be performed, "getenclavesignature".                                                        
 `enclavesignature`                        | `object` | The enclave signature of the order.                                                                                 
 &nbsp;&nbsp;`orderCommitment`     | `object` | The order commitment that the enclave signs.                                                         
 &nbsp;&nbsp;&nbsp;&nbsp;`transparent`| `object` | Transparent portion of the order commitment.           
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`side`| `number` | Side of the order, 0 for bid, 1 for ask.                                              
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`token` | `string` | Token address for the target project.                                                           
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`denomination` | `string` | Either the token address or USDC or ETH (set to 0x1 for this case).              
 &nbsp;&nbsp;&nbsp;&nbsp;`shielded`         | `string` | The shielded structure of the order commitment, i.e. the hash of the shielded portion of the raw order
 &nbsp;&nbsp;`signatureValue`     | `string` | The signature value.

**Example Request Payload:**
```javascript
const req = {
  "action": "sendorder",
  "data": {
    "transparent": {
      "side": 0,
      "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B", 
      "denomination": "0x1"
    },
    "shielded": {
      "price": 1000000000,
      "volume": 1000000000,
      "accessKey": "0x2a014355bdfe814d5f7c315b4e8b10c5a3446452c5c8d0593b9c59b7648fe3ed"
    }
  },
  "hash": "0x3d2b7e3d8a2e9409b878ba8dcfb9e9a9ebe0df7e7a3a2a918a6804d1b3cd4c19"
};

socket.send(JSON.stringify(req));
```

**Example Response Payload:**
```javascript
{
  "action": "sendorder",
  "enclaveSignature": {
    "orderCommitment": {
      "transparent": {
        "side": 0,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x3d2b7e3d8a2e9409b878ba8dcfb9e9a9ebe0df7e7a3a2a918a6804d1b3cd4c19"
    },
    "signatureValue": "0x2a014355bdfe814d5f7c315b4e8b10c5a3446452c5c8d0593b9c59b7648fe3ed"
  }
}
```

### Getting Crossed Orders
A client can request the orders that cross with an owned order commitment (Ō). We use the order commitment as a unique identifier for the order.

**Request Payload:**
| Field                | Type    | Description                                                                                     |
|----------------------|---------|-------------------------------------------------------------------------------------------------|
| `action`             | `string`| The action being performed, "getcrossedorders".                                                |
| `orderCommitment` | `object`| Order commitment to get crossed orders for.                 |

**Response Payload:**
| Field                | Type    | Description                                                                                     |
|----------------------|---------|-------------------------------------------------------------------------------------------------|
| `action`             | `string`| The action being performed, "getcrossedorders".                                                |
| `orderCommitment` | `object`| Order commitment for which the following orders array are crossed.                  |
| `orders`             | `array` | An array of crossed order objects. Each object contains details of the order.                   |

**Order Object:**

Each order object in the `orders` array has the following structure:

| Field                         |  Type     | Description                                                                                     
|-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
| `orderCommitment`             | `object` | Contains the order commitment details.                                                                                 
| &nbsp;&nbsp;`transparent`     | `object` | Contains transparent details of the order.                                                         
| &nbsp;&nbsp;&nbsp;&nbsp;`side`| `number` | Side of the order, 0 for bid, 1 for ask.                                                        
| &nbsp;&nbsp;&nbsp;&nbsp;`token` | `string` | Token address for the target project.                                                           
| &nbsp;&nbsp;&nbsp;&nbsp;`denomination` | `string` | Either the token address or USDC or ETH (set to 0x1 for this case).              
| &nbsp;&nbsp;`shielded`         | `string` | The shielded structure of the order commitment, i.e. the hash of the shielded portion of the raw order.

**Example Request Payload**
```javascript
{
  "action": "getcrossedorders",
  "orderCommitment": {
      "transparent": {
        "side": 0,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x3d2b7e3d8a2e9409b878ba8dcfb9e9a9ebe0df7e7a3a2a918a6804d1b3cd4c19"
    }
}
```

**Example Response Payload:**
```javascript
{
  "action": "getcrossedorders",
  "orderCommitment": {
      "transparent": {
        "side": 0,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x3d2b7e3d8a2e9409b878ba8dcfb9e9a9ebe0df7e7a3a2a918a6804d1b3cd4c19"
    },
  "orders": [
    {
      "orderCommitment": {
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x4e3c3e3e4c3d4b4c5b5a5b5c6d6e6f7e8a8b8c9d1a1b1c2d3e4f5g6h7i8j9k0l1m2"
    },
      "timestamp": 1678901234567,
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 500000000,
        "accessKey": "0x123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0" 
      }
    },
    {
      "orderCommitment": {
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c"
    },
      "timestamp": 1678901234578,
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 500000000,
        "accessKey": "0xe123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0"
      }
    }
  ]
}
```

### Notification of New Crossed Orders
A connected client will be notified of new orders that cross any of their open orders. 

**Publication Payload:**
| Field   | Type    | Description                                                                                     |
|---------|---------|-------------------------------------------------------------------------------------------------|
| `orderCommitment`             | `object` | Contains the order commitment details for the order.                                                    |
| `orders`| `array` | An array of crossed order objects. Each object contains details of the order.   

**Order Object:**

Each order object in the `orders` array has the following structure:

| Field                         |  Type     | Description                                                                                     
|-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
| `orderCommitment`             | `object` | Contains the order commitment details.                                                                                 
| &nbsp;&nbsp;`transparent`     | `object` | Contains transparent details of the order.                                                         
| &nbsp;&nbsp;&nbsp;&nbsp;`side`| `number` | Side of the order, 0 for bid, 1 for ask.                                                        
| &nbsp;&nbsp;&nbsp;&nbsp;`token` | `string` | Token address for the target project.                                                           
| &nbsp;&nbsp;&nbsp;&nbsp;`denomination` | `string` | Either the token address or USDC or ETH (set to 0x1 for this case).              
| &nbsp;&nbsp;`shielded`         | `string` | The shielded structure of the order commitment, i.e. the hash of the shielded portion of the raw order.


**Example Publication Payload:**
```javascript
{

  "orderCommitment": {
      "transparent": {
        "side": 0,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x3d2b7e3d8a2e9409b878ba8dcfb9e9a9ebe0df7e7a3a2a918a6804d1b3cd4c19"
    },
  "orders": [
    {
      "orderCommitment": {
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x4e3c3e3e4c3d4b4c5b5a5b5c6d6e6f7e8a8b8c9d1a1b1c2d3e4f5g6h7i8j9k0l1m2"
    },
      "timestamp": 1678901234567,
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 500000000,
        "accessKey": "0x123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0" 
      }
    },
    {
      "orderCommitment": {
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": "0x1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c"
    },
      "timestamp": 1678901234578,
      "transparent": {
        "side": 1,
        "token": "0xAb5801a7D398351b8bE11C439e05C5B3259aeC9B",
        "denomination": "0x1"
      },
      "shielded": {
        "price": 1000000000,
        "volume": 500000000,
        "accessKey": "0xe123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0"
      }
    }
  ]
}
```

## Clearing the Orderbook

**Note: This functionality is only available to the dev endpoint and is meant only for testing.**

To clear the orderbook, you need to send a request to the server. The server will then respond, confirming the action.

#### Request Payload

 Field                         |  Type     | Description                                                                                     
-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
 `action`                      | `string` | The action to be performed, "clearOrderbook". 

#### Response Payload

 Field                         |  Type     | Description                                                                                     
-------------------------------|---------------|:----------------------------------------------------------------------------------------------------------------------------------------------------------:
 `result`                      | `string` | The result of the request, "Orderbook cleared". 

#### Example Request Payload

```javascript
{
  "action": "clearOrderbook"
}
```
#### Example Response Payload

```javascript
{
  "result": "Orderbook cleared"
}
```
