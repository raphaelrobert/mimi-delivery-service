---
title: MIMI Delivery Service
abbrev: MIMI
docname: draft-robert-mimi-delivery-service-latest
category: info

ipr: trust200902
area: Security
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: R. Robert
    name: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com
 -  ins: K. Kohbrok
    name: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad.kohbrok@datashrine.de

--- abstract

This document describes the MIMI Delivery Service.

--- middle

# Introduction

The MLS protocol document specifies a protocol between two or more clients. The
MLS architecture document introduces an abstract concept of a "Delivery Service"
that is specifically responsible for ordering handshake messages and more
generally for delivering messages to the intended recipients.

This document describes a Delivery Service that performs the mandated ordering
of handshake messages but also offers the basic functionality that is common to
several messaging services. In particular, it offers the following features:

 * A protocol between clients and the Delivery Service that allows clients to
   interact with the Delivery Service, including the specific wire format of the
   messages. The protocol is specifically designed to be operated in a
   decentralized, federated architecture but can be used in single instances as
   well.
 * Assistance for new joiners of a group: The Delivery Service can keep and
   update state such as the public ratchet tree, group extensions, etc.
 * Message validation: The Delivery Service can inspect and validate handshake
   messages and reject malformed, invalid or malicious messages.
 * Built-in authentication of group members: The Delivery Service can
   authenticate group members, reject messages from non-members and enforce
   group administration policies. Additionally, the Delivery Service can be
   linked to the Authentication Service to enforce more policies and validation
   rules.
 * Privacy-preserving message delivery: The Delivery Service can deliver
   messages to the intended recipients without learning the identity of group
   members, in particular the sender or the recipients of messages. This feature
   is optional and is compatible with most other features, such as assistance
   and validation.
 * Scalability: The protocol makes no assumption about whether the Delivery Service
   runs as a single instance or as a cluster of instances.
 * Network fluid: While the Delivery Service would typically run as a
   server-side component, the only requirement is that it is accesible by all
   clients.
 * Transport agnostic: Messages between clients and the Delivery Service can be
   sent via an arbitrary transport protocol. Additionally, in the federated
   case, client messages to a guest Delivery Service can be forwarded by a
   client's local instance of this service.
 * Multi-device capable: The Delivery Service can be used by multiple devices
   of the same users and supports adding and removing devices of a user.
 * A Queueing Service subcomponent that can be used to implement a queueing
   service for messages that are delivered asynchronously, particularly in
   federated environments, as well as the protocol between the Delivery Service
   and the Queueing Service.

# Protocol overview

The Delivery Service is designed around an MLS group as a basic entity. The
Delivery Service can be used with multiple groups but does not require a link
between the groups.
All operations of the Delivery Service are relative to a specific group.

## High level functions of the Delivery Service

 * Authentication of group members
 * Validation of handshake messages
 * Ordering of handshake messages
 * Delivery of messages to the Queueing Service
 * Assistance for clients who (re)join a group

### Flow

The protocol is designed in a request/response pattern, whereby the client sends
a DSRequest message to the Delivery Service and the Delivery Service responds
with a DSResponse message. This pattern can easily be used over e.g. RESTful
APIs.

~~~ aasvg

Client           Delivery Service
|                |
| DSRequest      |
+--------------->|
|                |
| DSResponse     |
|<---------------+
|                |
~~~
{: title="Delivery Service Request/Response scheme" }

Depending on the client's request, the Delivery Service sends a message to the
Queueing Service. This happens whenever a message needs to be fanned out to all
other members of a group.

~~~ aasvg

Client           Delivery Service (Provider 1)     Delivery Service (Provider 2)
|                |                                 |
| DSRequest      |                                 |
+--------------->|                                 |
|                |                                 |
| DSResponse     |                                 |
|<---------------+                                 |
|                | Fanout                          |
|                +-------------------------------->|
|                |                                 |
~~~
{: title="Client/Delivery Service communication with fanout" }

In the federated case, messages to a guest Delivery Service can be proxied
via the sender's own Delivery Service.

## High level overview of the operations

 * Group creation/deletion
 * Update client fanout information
 * Join a group from a Welcome message as a new member
 * Join a group through an External Commit message as a new member
   or client of an existing member
 * Adding and removing users to/from a group
 * Adding and removing clients to/from a member of the group
 * Client updates (MLS leaf updates)
 * Resync a client with a group in case of state loss
 * Sending application messages

## Client removals induced by the Delivery Service

 The Delivery Service can remove clients from a group by issuing remove
 proposals in the following cases:

 * A user has removed a client from its account
 * A user has been deleted
 * The client is removed from the group because it has been inactive for too
   long

## Authentication

Clients authenticate to the Delivery Service using the signature key from their
respective leaf node in the group and a client-specific identifier.

Depending on the operation, one or more kind of client identifier can be used:

 * A leaf index: The client identifier is the leaf index of the client in the
   group.
 * KeyPackageRef: The client identifier is the KeyPackageRef of the client's
   KeyPackage.
 * User key hash: The client identifier is the hash of the user's public key.

The identifier is encoded as follows:

~~~
enum  {
  leaf_index(0),
  key_package_ref(1),
  user_key_hash(2),
} DSSenderType;

struct {
  DSSenderType sender_type;
  select (DSSenderID.sender_type) {
    case leaf_index
      uint32 leaf_index;
    case key_package_ref
      opaque key_package_ref<V>;
    case user_key_hash
      opaque user_key_hash<V>;
  }
} DSSenderID;
~~~

To authenticate, the client signs the corresponding request. The key used to
sign/verify a given request depends on the `sender_id`.

~~~

struct {
   DSSenderID sender_id;
   DSRequest request;
   // Signature over DSMessageTBS
   opaque signature<0..255>;
} DSMessage;

struct {
   DSSenderID sender_id;
   DSRequest request;
} DSMessageTBS;
~~~


## Request/Response scheme

The various request types, each corresponding to an operation, are combined as follows:

~~~
enum {
  ds_create_group(0),
  ds_delete_group(1),
  ...
} DSRequestType;
~~~

The request type is encoded in the DSRequest message as follows:

~~~
struct {
  DSRequestType request_type;
  select (DSRequest.request_type) {
    case ds_create_group:
      CreateGroupRequest create_group_request;
    case ds_delete_group:
      DeleteGroupRequest delete_group_request;
    ...
  }
} DSRequest;
~~~

# Operations

## Create group

A request from the client to create a new group on the Delivery Service.

The client sends the following CreateGroupRequest to the Delivery Service:

~~~
struct {
  opaque group_id<V>;
  KeyPackage key_package;
  ClientQueueConfig client_queue_config;
  GroupInfo group_info;
} CreateGroupRequest;
~~~

The Delivery Service responds with a CreateGroupResponse:

~~~
enum {
  invalid_group_id(0),
  invalid_key_package(1),
  invalid_group_info(2),
} CreateGroupResponse;
~~~

### Validation

The Delivery Service validates the request as follows:

 * The group ID is not empty and is not already in use.
 * The KeyPackage is valid, according to (I-D.ietf-mls-protocol) 13.4.3.2. Joining via
   External Commits.
 * The GroupInfo is valid, according to (I-D.ietf-mls-protocol) 11.1. KeyPackage
   validation.

##  Delete group

A request from the client to delete a group from the Delivery Service.

## Update queue information

A request from the client to update the queue information for a group. Clients
can provision a queue information object to indicate to the Delivery Service how
to transmit messages to the client during fanout.

## Join a group from a Welcome message

A request to retrieve information to join a group after having received a
Welcome message. The client requests the ratchet tree and the signed GroupInfo
from the Delivery Service.

```text
struct {
  opaque group_id<V>;
  SignatureKey sender;
  // TBD
} WelcomeInfo;
```

## Join a group through an External Commit message

A request to retrieve information to join a group, such as the ratchet tree and
the signed GroupInfo from the Delivery Service.

```text
struct {
  opaque group_id<V>;
  SignatureKey sender;
  // TBD
} ExternalCommitInfo;
```

# Add users

A request to add one or more users to a group. The users are added by their
respective KeyPackage in an MLS Add Proposal. The client create a Commit message
that covers the proposals and the corresponding Welcome message.

```text
struct {
  MLSMessage commit;
  MLSMessage welcome;
// TBD
} AddUsers;
```

## Remove user

A request to remove one or more users from a group. The users are removed by
their respective leaf node index from the group in an MLS Remove Proposal. The
client create a Commit message that covers the proposals .

```text
struct {
  MLSMessage commit;
// TBD
} RemoveUsers;
```

## Add a client to a group

A request to add one or more clients of an existing user to a group.

```text

struct {
  MLSMessage commit;
  MLSMessage welcome;
// TBD
} AddClients;
```

## Remove a client from a group

A request to remove one or more clients of an existing user from a group.

```text
struct {
  MLSMessage commit;
// TBD
} RemoveClients;
```

## Update client

A request to update the client's leaf node in the group. Regular updates are
important to get good Post-compromise Security (PCS) properties. The client
creates a Commit message that contains its new leaf node.

```text
struct {
  MLSMessage commit;
// TBD
} UpdateClient;
```

## Resync client

A request to resync the client with the group. This can be used to recover from
state loss. The client requests the ratchet tree and the signed GroupInfo from
the Delivery Service.

```text
struct {
  opaque group_id<V>;
  SignatureKey sender;
  // TBD
} ResyncClient;
```

## Update client fanout information

A request to update the client's fanout information. The client can use this to
update its fanout information, e.g. to change the queue information.

```text
struct {
  opaque group_id<V>;
  SignatureKey sender;
  // TBD
} UpdateClientFanoutInfo;
```

## Send message

A request to send a message to the group. The client sends an MLS application
message to the Delivery Service, which forwards it to the Queueing Service.

```text
struct {
  MLSMessage application_message;
  // TBD
} SendMessage;
```


# Delivery Service state

For each group hostend on a particular Delivery Service instance, the Delivery
Service keeps the following state:

 * The public MLS ratchet tree of the group
 * The GroupInfo of the current epoch
 * Past group states (for members who join through a Welcome message)
 * Client queue information (for message fanout via the Queueing Service)
 * Proposal store that stores standalone proposals of the current epoch
 * Extensions (TBD)

# Queueing Service

The Queueing Service is a service that is used by the Delivery Service to fan
out messages to clients. Each client has a queue that is used to store messages
for later retrieval by the client. The group state on the Delivery Service
contains client queue information for each client of a group. This information
is forwarded to the Queueing Service along with the message during message
fanout. The information is opaque to the Delivery Service and is only used by
the Queueing Service to deliver the message to the client.

Note that this document uses the term "client queue" to refer to an abstract
mechanism to forward messages to clients and does not necessarily imply that an
actual queueing mechanism is used.

This document only specifies the interface between the Delivery Service and the
Queueing Service. Other aspects of the Queueing Service are not specified in
this document.

## Queueing Service protocol

The Queueing Service protocol is a simple request/response protocol used between
the Delivery Service and the Queueing Service.

~~~ aasvg

Delivery Service                     Queueing Service
|                          |
| QueueingServiceRequest   |
+------------------------->|
|                          |
| QueueingServiceResponse  |
|<-------------------------+
|                          |
~~~
{: title="Queueing Service Request/Response scheme" }

Whenever the Delivery Service has validated an incoming message from a client
and starts to fan out the message, it sends the following request to the
Queueing Service for every client in the group:

~~~
struct {
  opaque client_queue_information<V>;
  MLSMessage message;
} QueueingServiceRequest;
~~~

The Queueing Service responds with a QueueingServiceResponse:

~~~
enum {
  ok(0),
  invalid_client_queue_information(1),
} QueueingServiceResponse;
~~~

In case the Queueing Service considers the client queue information to be in
invalid (e.g. because the corresponding user/client has been deleted), it
responds with invalid_client_queue_information. Otherwise, it responds with ok.

The Delivery Service can use rejected messages to subsequentially issue remove
proposals to remove the client from the group. The Delivery Service rejects
Commit messages and application messages until a client has sent a Commit that
covers the proposals.

## Queueing service in federated environments

In federated environments, the Queueing Service does not necessarily have to be
part of the same domain as the Delivery Service. In this case, the Delivery
Service sends requests over an inter-domain transport protocol to the Queueing
Service.

# Privacy preserving message delivery

If the desired mode of operation is for the Delivery Service to learn as little
as possible about the groups it serves and their members, the protocol can be
extended to protect the group state on the Delivery Service. This can happen
through two complementary mechanisms:

  * Unlinking member credentials from credentials issued by the Authentication
    Service, providing pseudonymity at the Delivery Service level
  * Encrypting the group state with a key that is only known to the group
    members, providing encryption at rest

In practice, the requests from clients to the Delivery Server are extended with
additional parameters, such as decryption keys for the group state and
additional pseudonymous user-level authentication.

# Extensions

The Delivery Service can make use of the following MLS extensions:

 * Example: Group administration extension
 * TBD
