---
title: Overview of Azure Event Hubs authentication and security model | Microsoft Docs
description: Event Hubs authentication and security model overview.
services: event-hubs
documentationcenter: na
author: sethmanheim
manager: timlt
editor: ''

ms.assetid: 93841e30-0c5c-4719-9dc1-57a4814342e7
ms.service: event-hubs
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 03/29/2017
ms.author: sethm;clemensv

---
# Event Hubs authentication and security model overview
The Azure Event Hubs security model meets the following requirements:

* Only clients that present valid credentials can send data to an Event Hub.
* A client cannot impersonate another client.
* A rogue client can be blocked from sending data to an Event Hub.

## Client authentication
The Event Hubs security model is based on a combination of [Shared Access Signature (SAS)](../service-bus-messaging/service-bus-sas.md) tokens and *event publishers*. An event publisher defines a virtual endpoint for an Event Hub. The publisher can only be used to send messages to an Event Hub. It is not possible to receive messages from a publisher.

Typically, an Event Hub employs one publisher per client. All messages that are sent to any of the publishers of an Event Hub are enqueued within that Event Hub. Publishers enable fine-grained access control and throttling.

Each Event Hubs client is assigned a unique token, which is uploaded to the client. The tokens are produced such that each unique token grants access to a different unique publisher. A client that possesses a token can only send to one publisher, but no other publisher. If multiple clients share the same token, then each of them shares a publisher.

Although not recommended, it is possible to equip clients and devices with tokens that grant direct access to an Event Hub. Any client that holds this token can send messages directly into that Event Hub. Such a client is not subject to throttling. Furthermore, the client cannot be blacklisted from sending to that Event Hub.

All tokens are signed with a SAS key. Typically, all tokens are signed with the same key. Clients are not aware of the key; this prevents other clients from manufacturing tokens.

### Create the SAS key
When creating an Azure Event Hubs namespace, the service generates a 256-bit SAS key named **RootManageSharedAccessKey**. This key grants send, listen, and manage rights to the namespace. You can create additional keys. It is recommended that you produce a key that grants send permissions to the specific Event Hub. For the remainder of this topic, it is assumed that you named this key **EventHubSendKey**.

The following example creates a send-only key when creating the Event Hub:

```csharp
// Create namespace manager.
string serviceNamespace = "YOUR_NAMESPACE";
string namespaceManageKeyName = "RootManageSharedAccessKey";
string namespaceManageKey = "YOUR_ROOT_MANAGE_SHARED_ACCESS_KEY";
Uri uri = ServiceBusEnvironment.CreateServiceUri("sb", serviceNamespace, string.Empty);
TokenProvider td = TokenProvider.CreateSharedAccessSignatureTokenProvider(namespaceManageKeyName, namespaceManageKey);
NamespaceManager nm = new NamespaceManager(namespaceUri, namespaceManageTokenProvider);

// Create Event Hub with a SAS rule that enables sending to that Event Hub
EventHubDescription ed = new EventHubDescription("MY_EVENT_HUB") { PartitionCount = 32 };
string eventHubSendKeyName = "EventHubSendKey";
string eventHubSendKey = SharedAccessAuthorizationRule.GenerateRandomKey();
SharedAccessAuthorizationRule eventHubSendRule = new SharedAccessAuthorizationRule(eventHubSendKeyName, eventHubSendKey, new[] { AccessRights.Send });
ed.Authorization.Add(eventHubSendRule); 
nm.CreateEventHub(ed);
```

### Generate tokens
You can generate tokens using the SAS key. You must produce only one token per client. Tokens can then be produced using the following method. All tokens are generated using the **EventHubSendKey** key. Each token is assigned a unique URI.

```csharp
public static string SharedAccessSignatureTokenProvider.GetSharedAccessSignature(string keyName, string sharedAccessKey, string resource, TimeSpan tokenTimeToLive)
```

When calling this method, the URI should be specified as `//<NAMESPACE>.servicebus.windows.net/<EVENT_HUB_NAME>/publishers/<PUBLISHER_NAME>`. For all tokens, the URI is identical, with the exception of `PUBLISHER_NAME`, which should be different for each token. Ideally, `PUBLISHER_NAME` represents the ID of the client that receives that token.

This method generates a token with the following structure:

```csharp
SharedAccessSignature sr={URI}&sig={HMAC_SHA256_SIGNATURE}&se={EXPIRATION_TIME}&skn={KEY_NAME}
```

The token expiration time is specified in seconds from Jan 1, 1970. The following is an example of a token:

```csharp
SharedAccessSignature sr=contoso&sig=nPzdNN%2Gli0ifrfJwaK4mkK0RqAB%2byJUlt%2bGFmBHG77A%3d&se=1403130337&skn=RootManageSharedAccessKey
```

Typically, the tokens have a lifespan that resembles or exceeds the lifespan of the client. If the client has the capability to obtain a new token, tokens with a shorter lifespan can be used.

### Sending data
Once the tokens have been created, each client is provisioned with its own unique token.

When the client sends data into an Event Hub, it tags its token with the send request. To prevent an attacker from eavesdropping and stealing the token, the communication between the client and the Event Hub must occur over an encrypted channel.

### Blacklisting clients
If a token is stolen by an attacker, the attacker can impersonate the client whose token has been stolen. Blacklisting a client renders that client unusable until it receives a new token that uses a different publisher.

## Authentication of back-end applications
To authenticate back-end applications that consume the data generated by Event Hubs clients, Event Hubs employs a security model that is similar to the model that is used for Service Bus topics. An Event Hubs consumer group is equivalent to a subscription to a Service Bus topic. A client can create a consumer group if the request to create the consumer group is accompanied by a token that grants manage privileges for the Event Hub, or for the namespace to which the Event Hub belongs. A client is allowed to consume data from a consumer group if the receive request is accompanied by a token that grants receive rights on that consumer group, the Event Hub, or the namespace to which the Event Hub belongs.

The current version of Service Bus does not support SAS rules for individual subscriptions. The same holds true for Event Hubs consumer groups. SAS support will be added for both features in the future.

In the absence of SAS authentication for individual consumer groups, you can use SAS keys to secure all consumer groups with a common key. This approach enables an application to consume data from any of the consumer groups of an Event Hub.

## Next steps
To learn more about Event Hubs, visit the following topics:

* [Event Hubs overview]
* [Overview of Shared Access Signatures]
* [Sample applications that use Event Hubs]

[Event Hubs overview]: event-hubs-what-is-event-hubs.md
[Sample applications that use Event Hubs]: https://github.com/Azure/azure-event-hubs/tree/master/samples
[Overview of Shared Access Signatures]: ../service-bus-messaging/service-bus-sas.md

