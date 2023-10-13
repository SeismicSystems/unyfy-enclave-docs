---
title: Elliptic Enclave WebSocket API Documentation
description: This document provides an overview and instructions on how to interact with the Elliptic Enclave WebSocket API.
---

## Overview

The Elliptic Enclave API is a WebSocket API that allows clients to interact with the enclave used for Elliptic's on-chain dark pool solution. This document provides detailed information on how to establish a connection, send messages, and place orders using this API.

## General Considerations

The API uses Amazon API Gateway to route requests to the backend EC2 instances running the Nitro enclaves, powered by Amazon EKS. This is done using a REST API which is abstracted away from the client.

### Establishing a Connection

To initiate a connection to the API, use the following command. Upon connection to the API, API Gateway triggers a route that calls a REST API that stores your unique connection ID.

Here is an example of how to establish a connection using JavaScript:

```javascript
const socket = new WebSocket('wss://abcdef123.execute-api.us-west-2.amazonaws.com/production');

socket.addEventListener('open', function (event) {
    console.log('Connected');
});

socket.addEventListener('message', function (event) {
    console.log('Message from server ', event.data);
});
```


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
}


```

#### getenclavesignature

**Response.** Server's response after receiving a client's order through through ```sendorder```. The server returns an enclave signature. 


#### getcrossedorders

**Publication.** The enclave publishes the crossed orders.

**Payload:**
| Field                | Type    | Description                                                                                     |
|----------------------|---------|-------------------------------------------------------------------------------------------------|
| `action`             | `string`| The action being performed, "getcrossedorders".                                                |
| `matchedWithOrderId` | `string`| The unique identifier of the order for which these orders are crossed/matched.                  |
| `orders`             | `array` | An array of crossed order objects. Each object contains details of the order.                   |

Example of payload:

```javascript
{
  "action": "getcrossedorders",
  "matchedWithOrderId": "order12345",
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