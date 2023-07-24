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
(DS) that is specifically responsible for ordering handshake messages and more
generally for delivering messages to the intended recipients.

This document describes a Delivery Service that performs the mandated ordering
of handshake messages and uses MLS to implement a variety of other features:

 * A protocol between clients and the Delivery Service that allows clients to
   interact with the Delivery Service, including the specific wire format of the
   messages. The protocol is specifically designed to be operated in a
   decentralized, federated architecture but can be used in single instances as
   well.
 * Discovery and connection establishment between clients: Clients can request
   key material to establish an end-to-end encrypted channel to other clients.
 * Assistance for new joiners of a group: The Delivery Service can keep and
   update state such as the public ratchet tree, group extensions, etc.
 * Message validation: The Delivery Service can inspect and validate handshake
   messages and reject malformed, invalid or malicious messages.
 * Built-in authentication of group members and policy enforcement: The Delivery
   Service can authenticate group members, reject messages from non-members and
   enforce group administration policies. Additionally, the Delivery Service can
   be linked to the Authentication Service to enforce more policies and
   validation rules.
 * Scalability: The protocol makes no assumption about whether the Delivery Service
   runs as a single instance or as a cluster of instances.
 * Network fluid: While the Delivery Service would typically run as a
   server-side component, the only requirement is that it is accessible by all
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

TODO: Make use of MUST/SHOULD, etc. throughout.

# Terminology

This document uses the terminology defined in {{!I-D.ietf-mls-protocol}} with
the following additions:

* DS Domain: Fully qualified domain name as defined in RFC 2181 that represents
  the DS. This does not necessarily have to be the domain under which the DS is
  reachable on the network. Discovering that domain is the responsibility of the
  MIMI transport protocol over which the MIMI DS protocol runs.
TODO: User/client/identifiers definitions are preliminary.
* User: The operator of one or more clients, identified by its
  user identifier.
* User identifier: An identifier that is unique in the scope of its DS and that
  allows resolution to the providers DS domain.
* Client: An MLS client with a unique client identifier.
* Client identifier: An octet string that uniquely identifies the client. A
  client identifier includes the identifier of its user or otherwise allows the
  association between the client and its user.
* Connection: An agreement between two users, where each user authorizes the
  other to send them messages and to add them to groups.
* Connection establishment: The process by which a connection between two users
  is established. The process is manually initiated by one user and involves the
  discovery of the other user. The other user must be able to authenticate the
  initiator and manually authorize the response.
* Connection group: An MLS group consisting of the clients of two users. For
  each pair of users, there can only be one connection group between them. A
  connection group is created when one user requests a connection with another
  user, followed by that user's consent.

# Architecture and protocol overview

The MIMI DS protocol allows interoperability between an owning DS which hosts a
group conversation and one or more guest DSs which are home to one or more of
the conversation's group members. Underlying each group conversation is an MLS
group that facilitates end-to-end encryption and authentication between group
members.

The main purpose of the MIMI DS protocol thus is to ensure that guest clients
can participate in the group. With MLS as the underlying protocol, this means
that the MIMI DS protocol is primarily concerned with the fan-out of MLS
messages (both from and to guest clients), as well the assistance of guest
clients in joining MLS groups.

The MIMI DS protocol requires clients to send MLS messages as PublicMessages
(with the exception of messages with content type `application`). This allows
the owning DS to track the MLS group state in the same way as a client would,
enabling it to keep an up-to-date and fully authenticated list of members, as
well as provide the full MLS group state to joining group members, even for
those joining via external commit. In addition, the DS can verify messages and
enforce access control policies on group operations.

## Client to server and server to server protocol {#c2s}

MLS being a protocol for end-to-end encryption, a subset of MIMI DS protocol
messages have to originate from the clients rather than the interoperating
delivery services.

The MIMI DS protocol consists of two parts: A client-to-server part that allows
guest clients to interact with the owning DS of one of their groups and a
server-to-server protocol that allows an owning DS to fan out messages to guest
DSs, which can subsequently store and forward messages to their respective
clients.

Note that the client-to-server part of the protocol can optionally be proxied
via the guest DS of the sending client.

## Transport for the MIMI DS protocol

The MIMI DS protocol requires a transport protocol that provides confidentiality
of messages and that allows the discovery of a DS based on the DS domain in a
user identifier.

Both the client-to-server part, as well as the server-to-server part of the MIMI
DS protocol provide sender authentication, leaving recipient authentication as
provided, for example, by the HTTPS protocol to the transport layer.

TODO: If the transport layer provides mutual authentication, at least the
server-to-server part of the MIMI DS protocol can be changed accordingly.

In the event that a guest DS proxies the client-server part of the MIMI DS
protocol, the transport protocol can be used to facilitate additional
functionality relevant to server-to-server communication, such as e.g.
server-to-server authentication.

## Flow

~~~aasvg
+-------------+ DSRequest    +--------------+
|             +------------->+              |
| Sending     |              | Owning DS    |
| Client      |              |              |
|             +<-------------+              |
+-------------+ DSResponse   +--+--------+--+
                                |        ^
                DSFanoutRequest |        | DSFanoutResponse
                                v        |
                             +--+--------+--+
                             |              |
                             | Guest DS     |
                             |              |
                             |              |
                             +--------------+
~~~
{: #full-sending-flow title="Architecture overview" }

~~~aasvg
+-------------+              +--------------+
|             +------------->+              |
| Sending     | (proprietary | Guest DS     |
| Client      |   protocol)  | (proxy)      |
|             +<-------------+              |
+-------------+              +--+--------+--+
                                |        ^
                      DSRequest |        | DSResponse
                                v        |
                             +--+--------+--+
                             |              |
                             | Owning DS    |
                             |              |
                             |              |
                             +--+--------+--+
                                |        ^
                DSFanoutRequest |        | DSFanoutResponse
                                v        |
                             +--+--------+--+
                             |              |
                             | Guest DS     |
                             |              |
                             |              |
                             +--------------+
~~~
{: #proxied-sending-flow title="Alternative with a guest DS as proxy" }

{{full-sending-flow}} and {{proxied-sending-flow}} show example protocol flows,
where a client sends a request to the owning DS, followed by the owning DS
fanning out a message to a guest DS. In {{proxied-sending-flow}}, the request
sent by the client is proxied by the guest DS of that client.

For the remainder of this document, we assume that clients send requests
directly to the owning DS. Proxying the requests over the client's own DS can
always be done and does not change the functionality.

Both the message sending and the fanout parts of the protocol are designed in a
request/response pattern. In the first protocol part, the client sends a
DSRequest message to the Delivery Service and the Delivery Service responds with
a DSResponse message. This pattern can easily be used over e.g. RESTful APIs.

~~~aasvg
Client           Owning Delivery Service
|                |
| DSRequest      |
+--------------->|
|                |
| DSResponse     |
|<---------------+
|                |
~~~
{: title="Delivery Service Request/Response scheme" }

For the second part of the protocol the Delivery Service sends a DSFanoutRequest
to each guest DS. This happens whenever a message needs to be fanned out to all
other members of a group as a result of an incoming DSRequest. The guest DS in
turn responds with a DSFanoutResponse.

~~~aasvg
Client           Owning Delivery Service           Guest Delivery Service
|                |                                 |
| DSRequest      |                                 |
+--------------->|                                 |
|                |                                 |
| DSResponse     |                                 |
|<---------------+                                 |
|                | DSFanoutRequest                 |
|                +-------------------------------->|
|                |                                 |
|                | DSFanoutResponse                |
|                |<--------------------------------+
|                |                                 |
~~~
{: title="Client/Delivery Service communication with fanout" }

## Supported operations

The MIMI DS protocol allows guest clients to request a variety of operations for
example to manage group membership, join groups, or send messages.

 * Group creation/deletion
 * Join a group from a Welcome message as a new member
 * Join a group through an External Commit message as a new member
   or client of an existing member (e.g. Resync a client after state loss)
 * Adding and removing users to/from a group
 * Adding and removing clients to/from a member of the group
 * Client updates (MLS leaf updates)
 * Sending application messages
 * Download of KeyPackages
 * Enqueueing of a message fanned out from an owning DS
 * Discovery of users and their clients
 * Download of connection-specific KeyPackages

## Client removals induced by the Delivery Service

Independent of client requests, the Delivery Service itself can remove clients
from a group by issuing remove proposals in the following cases:

* A user has removed a client from its account
* A user has been deleted
* The client is removed from the group because it has been inactive for too
  long

## Serialization format

MLS messages use the presentation language and encoding format defined in
{{!RFC8446}} with the extensions defined in the MLS protocol specification. The
MIMI DS protocol uses the same serialization format, as both clients and DS
already have to support it to process MLS messages.

Octet strings resulting from serialization in this format are unambiguous and
require no further canonicalization.

## KeyPackages

Clients have to upload KeyPackages such that others can add them to groups. In
the context of interoperability, this means that clients have to be able to
download KeyPackages of clients belonging to other DSs.

The MIMI DS protocol allows clients to download the KeyPackages of other
clients. Uploading KeyPackages is outside of the scope of this protocol, as it
is not relevant for interoperability.

TODO: KeyPackages of last resort should be marked. Ideally by using a KeyPackage
extension.

### Connection KeyPackages

In addition to regular KeyPackages, the MIMI DS protocol requires clients to
provide connection KeyPackages for other clients to download. Connection
KeyPackages are meant for use in the connection establishment process (explained
in more detail in {{connection-establishment-flow}}) and differ from regular
KeyPackages only in that they include a LeafNode extension marking them as such.

TODO: Such an extension would have to be specified in the context of the MLS WG.

## Clients and users

TODO: This section needs to be revisited once identifiers and client/user
authentication in general have been decided on by the MIMI WG.

Secure messaging applications typically have a notion of users, where each user
has one or more clients. As MLS does not have a native notion of users, it has
to be provided by the MIMI DS protocol.

The MIMI DS protocol only requires that the user identifier can be derived from
the client identifier. Authentication of a client is then assumed to imply
authentication of the user.

## Version agility

MLS provides version, ciphersuite and extension agility. The versions,
ciphersuites and extensions a client supports are advertised in its LeafNodes,
both in all of the client's groups, as well as in its KeyPackages through the
Capabilities field (Section 7.2 of {{!I-D.ietf-mls-protocol}}).

MLS DS protocol clients MUST make use of a LeafNode extension to advertise the
MIMI DS protocol versions they support.

TODO: Such an extension would have to be specified in the context of the MLS WG.

## Group lifecycle

Upon creation MLS groups are parameterized by a GroupID, an MLS version number,
as well as a ciphersuite. All three parameters are fixed and cannot be changed
throughout the lifetime of a group.

Groups capable of interoperability with the MIMI DS protocol MUST use a
GroupContext extension that indicates the MIMI DS protocol version with which it
was created. This extension MUST NOT be changed throughout the lifetime of the
group.

While all these parameters cannot be changed throughout a group's lifetime, the
group can be re-initialized as described in Section 11.2. of
{{!I-D.ietf-mls-protocol}} to create a new group with a new set of parameters.

The MIMI DS protocol supports re-initializations of groups using the
corresponding ReInitialization operation under the condition that all MLS
parameters are compatible with the MIMI DS protocol version.

If a group is no longer used, it can be deleted either by a client or the DS
itself.

TODO: Each MIMI DS protocol version should probably fix a set of ciphersuites,
MLS protocol versions and maybe even extensions it supports. New ones can be
added with protocol version upgrades.

## Framing and processing overview

### Client to server requests

All client to server requests consist of a MIMI DS specific protocol wrapper
called DSRequst. DSRequest contains the MIMI DS protocol version, a body with
operation-specific data, as well as authentication information.

~~~
enum {
  ds_request_group_id(0),
  ds_create_group(1),
  ds_delete_group(2),
  ...
} DSRequestType;

struct {
  DSRequestType request_type;
  select (DSRequestBody.request_type) {
    case ds_delete_group:
      DeleteGroupRequest delete_group_request;
    ...
  }
} DSRequestBody;

struct {
  DSProtocolVersion version;
  DSRequestBody request_body;
  DSAuthData authentication_data;
} DSRequest;
~~~

A DS supports a variety of operations. For presentational reasons, we only
define DSRequestType and the corresponding `case` statement in DSRequestBody
partially here. The full definition with all operations relevant for the
normal DS operating mode can be found in {{operations}}.

The `authentication_data` field of a DSRequest depends on the request type and
contains the data necessary for the DS to authenticate the request.

~~~
enum {
  Anonymous,
  ClientSignature,
} DSAuthType;

struct {
  DSAuthType auth_type;
  select (DSAuthData.auth_type) {
    case Anonymous:
      struct {};
    case ClientSignature:
      uint32 sender_index;
      opaque signature<0..255>;
  }
} DSAuthData;
~~~

Before the DS performs the requested operation, it performs an authentication
operation depending on the DSAuthType.

* Anonymous: No authentication required
* ClientSignature: The DS uses the public signature key of the MLS client in
  leaf with the leaf index `sender_index` to verify the signature over the
  following struct:

~~~
struct {
  DSProtocolVersion protocol_version;
  DSRequestBody request_body;
  u32 sender_index
} ClientSignatureTBS
~~~

Note that all group operations additionally contain an MLSMessage the content
of which mirrors the request type, e.g., an AddUsers request wraps an MLS commit
that in turn contains the Add proposals for the users' clients. In that case,
the DS uses the GroupID inside the MLSMessage to determine which group the
request refers to and verifies the MLSMessage in the same way an MLS client
would (including the signature).

Depending on the nature of the request, clients can also include data in the AAD
field of the MLSMessage, where it can be read and authenticated by both DS and
all other group members.

After performing the desired operation using the data in DSRequestBody the DS
responds to the client (or the proxying guest DS) with a DSResponse.

~~~

struct {
  DSProtocolVersion protocol_version;
  DSResponseBody response_body;
} DSResponse

enum DSResponseType {
  Ok,
  Error,
  GroupID,
  WelcomeInfo,
  ExternalCommitInfo
  SignaturePublicKey,
  KeyPackages,
  ConnectionKeyPackages,
}

struct DSResponseBody {
  DSResponseType response_type;
  select (DSResponseBody.response_type) {
    case Ok:
      struct {};
    case Error:
      DSError error;
    case GroupID:
      opaque group_id<V>;
    case WelcomeInfo:
      optional<Node> ratchet_tree<V>;
    case ExternalCommit:
      MLSMessage: group_info;
      optional<Node> ratchet_tree<V>;
    case SignaturePublicKey:
      SignaturePublicKey signature_public_key;
    case KeyPackages:
      KeyPackage key_packages<V>;
    case ConnectionKeyPackages:
      KeyPackage key_packages<V>;
  }
}

struct {
  TODO: Operation specific errors.
} DSError
~~~

### Server to server requests

After sending the response, and depending on the operation the DS might fan out
messages to one or more guest DSs.

To that end, it wraps the MLSMessage to be fanned out into a DSFanoutRequest.
In addition to the MLSMessage, the DSFanoutRequest contains the protocol
version, the FQDN of the sending DS and the identifiers of all clients on DS
that the message is meant to be fanned out to.

If the message needs to be fanned out to more than one guest DS, the DS prepares
different messages for each destination DS with each message containing only the
DSFanoutRecipients of that DS.

~~~
struct {
  DSProtocolVersion protocol_version;
  FQDN sender;
  opaque recipient_ids<V>;
  MLSMessage mls_message;
  // Signature over the above fields
  opaque signature<0..255>;
} DSFanoutRequest
~~~

The DS receiving a DSFanoutRequest can then store and forward the contained MLS
message to the clients indicated in the `recipient_ids` field.

The receiving DS verifies the signature using the sending DS' public signature
key, process the message and sends a DSFanoutResponse.

~~~
enum {
  Ok,
  Error,
} DSFanoutResponseType

struct {
  TODO: Fanout error types
} DSFanoutError

struct DSFanoutResponseBody {
  DSFanoutResponseType response_type;
  select (DSFanoutResponseBody.response_type) {
    case Ok:
      struct {};
    case Error:
      DSFanoutError error;
  }
}

struct {
  DSProtocolVersion protocol_version;
  DSResponseBody response_body;
} DSFanoutResponse
~~~

### KeyPackages

As noted in {{keypackages}}, clients must upload KeyPackages such that other
clients can add them to groups.

## Connection establishment flow

A user can establish a connection to another user by creating a connection
group, fetching a connection KeyPackage of the target user's clients from its DS
and inviting that user to the connection group. The receiving user can process
the Welcome and figure out from the group state who the sender is. Additional
information can either be attached to the group via an extension, or via MLS
application messages within that group.

TODO: This is a sketch for a simple connection establishment flow that allows
the recipient to identify and authenticate the sender and that establishes an
initial communication channel between the two users. More functionality can be
added via additional messages or MLS extensions.

~~~aasvg
Alice                        Owning DS                 Guest DS            Bob
+                            +                         +                   +
| Request Connection         |                         |                   |
| KeyPackages                |                         |                   |
+----------------------------------------------------->+                   |
|                            |                         |                   |
| Connection KeyPackages     |                         |                   |
+<-----------------------------------------------------+                   |
|                            |                         |                   |
| Create Connection group    |                         |                   |
+--------------------------->+                         |                   |
|                            |                         |                   |
| Add Responder              |                         | Deliver Welcome   |
+--------------------------->+ Fan out Welcome         | (proprietary      |
|                            +------------------------>+  protocol)        |
|                            |                         +------------------>+
|                            |                         |                   |
+                            +                         +                   +

~~~
{: title="Example flow for connection establishment" }


# DS assisted joining

To verify and deliver messages, authenticate clients as members of a group and
to assist clients that want to join a group, the DS keeps track of the state of
each group it owns. More specifically, it keeps track of the group's ratchet
tree, the group's GroupContext and other information required to produce a valid
GroupInfo for the current group epoch. It does this by processing incoming MLS
messages in the same way a member of that group would, except of course that the
DS doesn't hold any private key material.

While MLS messages are sufficient to keep track of most of the group information,
it is not quite enough to create a GroupInfo. To allow the DS to provide a valid
GroupInfo to externally joining clients, it additionally requires clients to
provide the remaining required information. Concretely, it requires clients to
upload the Signature and the GroupInfo extensions. Clients need to send this
information whenever they send an MLS message (i.e. an MLSMessage struct) that
contains a commit.

~~~
struct {
  Extension group_info_extensions<V>;
  opaque Signature<V>;
} PartialGroupInfo
~~~

The combination of a commit and a partial group info is called an
MLSGroupUpdate.

~~~
struct {
  MLSMessage commit;
  PartialGroupInfo partial_group_info;
} MLSGroupUpdate
~~~

Whenever the DS receives an MLSGroupUpdate, it must verify that the
MLSMessage contains a PublicMessage with a commit and that commit and partial
group info are valid relative to the existing group state according to the MLS
specification.

TODO: For now there is no distinct endpoint to obtain authentication material
that allows the DS to authenticate clients. This would be part of the AS design.

By tracking the group information in this way, the DS can help clients that join
via external commit by providing them with a ratchet tree and a group into.

Similary, clients that wish to join a group via a regular invite (i.e. a Welcome
message) have already received a GroupInfo and can obtain a ratchet tree from
the DS.

In the time between a client being added to a group by a commit and the client
wanting to join the group, the group state can have progressed by one or more
epochs. As a consequence, the DS MUST keep track of epochs in which clients are
added and store the corresponding group states until each client has
successfully joined.

# Handling of standalone MLS Proposals {#proposals-by-reference}

MLS relies on a proposal-commit logic, where the proposals encode the specific
action the sending client intends to take and the commit then performs the
actions of a set of commits.

The advantage of this approach is that the sender of the proposal does not have
to be the committer, which allows, for example, the DS to propose the removal of
a client, or a client to propose that it be removed from the group. Note that
the latter example is the only way that a client can remove itself (i.e. leave)
from a group.

In MLS proposals can be committed "by reference" (if the proposal was sent
separately before the commit), or "by value" if the proposal is sent as part of
the commit itself.

In all operations specified in the follow sections, the proposals in the commit
that is included in the DSRequest MUST match the semantics of the operation and
all of those proposals MUST be committed by value. For example, the commit
in the AddUsersRequest MUST only contain Add proposals.

However, in addition, each commit MAY also include an arbitrary number of valid
proposals that were sent previously in the same epoch, such as server-initiated
Remove proposals, or proposals sent as part of a self-remove operation.

Such additional proposals MUST be committed by reference.

To allow the DS to send proposals, all groups MUST contain an `external_senders`
extension as defined in Section 12.1.8.1. of {{!I-D.ietf-mls-protocol}} that
includes the DS' credential and its signature public key.

TODO: Details of the DS credential. A BasicCredential with the FQDN of the DS
would probably be sufficient.

TODO: Proposals by reference pose a problem in the context of external commits,
as, even if the external committer had access to all proposals in an epoch, it
wouldn't be able to verify them, thus potentially leading to an invalid external
commit. A solution could be introduced either as part of the MIMI DS protocol,
or as an MLS extension. The latter would be preferable, as other users of MLS
are likely going to encounter the same problem.

# Operations

The DS supports a number of operations, each of which is represented by a
variant of the DSRequestType enum and has its own request body.

~~~
enum {
  ds_request_group_id(0),
  ds_create_group(1),
  ds_delete_group(2),
  ds_add_users(3),
  ds_remove_users(4),
  ds_add_clients(5),
  ds_remove_clients(6),
  ds_self_remove_client(7),
  ds_update_client(8),
  ds_external_join(9),
  ds_send_message(10),
  ds_signature_public_key(11),
  ds_key_packages(12),
  ds_connection_key_packages(13),
  ds_re_initialize_group(14),
} DSRequestType;

struct {
  DSRequestType request_type;
  select (DSRequestBody.request_type) {
    case ds_request_group_id:
      struct {};
    case ds_create_group:
      CreateGroupRequest create_group_request;
    case ds_delete_group:
      DeleteGroupRequest delete_group_request;
    case ds_add_users:
      AddUsersRequest add_users_request;
    case ds_remove_users:
      RemoveUsersRequest remove_users_request;
    case ds_add_clients:
      AddClientsRequest add_clients_request;
    case ds_remove_clients:
      RemoveClientsRequest remove_clients_request;
    case ds_self_remove_clients:
      SelfRemoveClientsRequest self_remove_clients_request;
    case ds_update_client:
      UpdateClientRequest update_client_request;
    case ds_external_join:
      ExternalJoinRequest external_join_request;
    case ds_send_message:
      SendMessageRequest send_message_request;
    case ds_signature_public_key:
      struct {};
    case ds_key_packages:
      KeyPackagesRequest key_packages_request;
    case ds_connection_key_packages:
      ConnectionKeyPackagesRequest connection_key_packages_request;
    case ds_re_initialize_group:
      ReInitializeGroupRequest re_initialize_group_request;
  }
} DSRequestBody;
~~~

## Request group id

Clients can use this operation to request a group id. This group ID can
subsequently used to create a group on this DS.

After receiving this request, the DS generates a unique group id and responds
with a DSResponse struct of type GroupID.

## Create group

A request from the client to create a new group on the Delivery Service. This
operation can be used both for the creation of regular groups and for the
creation of connection groups.

The client sends the following CreateGroupRequest to the Delivery Service:

~~~
struct {
  opaque group_id<V>;
  LeafNode LeafNode;
  MLSMessage group_info;
} CreateGroupRequest;
~~~

The Delivery Service internally creates and stores the group based on the
information in the request and responds with a CreateGroupResponse:

~~~
enum {
  invalid_group_id(0),
  invalid_leaf_node(1),
  invalid_group_info(2),
} CreateGroupResponse;
~~~

TODO: How to mark connection groups? Have the creator include a connection
extension in the LeafNode?

**Validation:**

The Delivery Service validates the request as follows:

 * The group ID MUST NOT be not empty and MUST NOT already be in use.
 * The LeafNode MUST be valid, according to {{!I-D.ietf-mls-protocol}} 7.3. Leaf
   Node validation.
 * The GroupInfo MUST be valid, according to {{!I-D.ietf-mls-protocol}} 12.4.3.
   Adding Members to the Group.

## Delete group

A request from the client to delete a group from the Delivery Service. This
operation allows clients to delete a group from the DS. Clients can of course
keep a copy of the group state locally for archival purposes.

~~~
struct {
  MLSGroupUpdate group_update;
} DeleteGroupRequest;
~~~

**Validation:**

The Delivery Service validates the request as follows:

 * The MLSGroupUpdate MUST contain a PublicMessage with a commit that contains
   Remove proposals for every member of the group except the committer.


## Add users to a group

A request from a client to add one or more users to a group. For each user, one
or more of the user's clients are added to the group. The Welcome messages in
the request are then fanned out to the user's DSs.

To obtain the KeyPackages required to add the users' clients, the sender must
first fetch the clients' KeyPackages from their DS.

~~~
struct {
  MLSGroupUpdate group_update;
  MLSMessage welcome_messages<V>;
} AddUsersRequest;
~~~

**Validation:**

* The MLSGroupUpdate in the `commit` field MUST contain a PublicMessage with a
  commit that contains only Add proposals with the possible exception of Remove
  proposals as detailed in {{proposals-by-reference}}.
* The commit MUST NOT change the sender's client credential.
* Add proposals MUST NOT contain clients of existing group members.
* Add proposals MUST NOT contain connection KeyPackages, except if the group is
  a connection group.
* If guest users are added as part of the request, there MUST be a distinct
  Welcome message for each guest DS involved.


## Remove users from a group

A request from a client to remove one or more users from a group. The DS will
still fan out the request to the users. The commit contained in the message will
allow the users to recognize that they were removed.

~~~
struct {
  MLSGroupUpdate group_update;
} RemoveUsersRequest;
~~~

**Validation:**

* The MLSGroupUpdate MUST contain a PublicMessage with a commit that contains
  only Remove proposals (see also {{proposals-by-reference}}).
* The commit MUST NOT change the sender's client credential.
* The Remove proposals that are committed by-value MUST always remove all
  clients of one or more users.

## Add clients to a group

A request from a client to add one or more clients of the same user. This
operation allows users to add new clients to an existing group. Alternatively,
new clients can add themselves by joining via external commit.

~~~
struct {
  MLSGroupUpdate group_update;
  MLSMessage welcome_messages<V>;
} AddClientsRequest;
~~~

**Validation:**

* The MLSGroupUpdate MUST contain a PublicMessage with a commit that contains
  only Add proposals with the possible exception of Remove proposals as detailed
  in {{proposals-by-reference}}.
* The commit MUST NOT change the sender's client credential.
* All Add proposals MUST contain clients of the same user as an existing group
  member.

## Remove clients from a group

A request from a client to remove one or more other clients of the same user
from a group. This operation allows users to remove their own clients from a
group. Note that this operation cannot be used by a client to remove itself from
the group. For that purpose, the SelfRemoveClientRequest should be used instead.

~~~
struct {
  MLSGroupUpdate group_update;
} RemoveClientsRequest;
~~~

**Validation:**

* The MLSGroupUpdate MUST contain a PublicMessage with a commit that contains only
  Remove proposals.
* The commit MUST NOT change the sender's client credential.
* All Remove proposals committed to by-value MUST target clients of the same
  user as the sending client

## Self remove a client from a group

A request from a client to remove itself from the group. If it's the last client
of a user, this effectively removes the user from the group. Note that this
request only contains a proposal, so the client is not effectively removed from
the group until another group member commits that proposal. See
{{proposals-by-reference}} for more details.

~~~
struct {
  MLSMessage proposal;
} SelfRemoveClientRequest;
~~~

**Validation:**

* The MLSMessage MUST contain a PublicMessage that contains a single
  Remove proposal.
* The Remove proposal MUST target the sending client.

## Update a client in a group

A request from a client to update its own leaf in an MLS group. This operation
can be used to update any information in the sender's leaf node. For example,
the sender could use this operation to update its key material to achieve
post-compromise security, update its Capabilities extension, or its leaf
credential.

~~~
struct {
  MLSGroupUpdate group_update;
} UpdateClientRequest;
~~~

**Validation:**

* The MLSGroupUpdate MUST contain a PublicMessage that contains a commit with an
  UpdatePath, but no other proposals by value.
* If the leaf credential is changed by the update, the DS MUST validate the new
  credential.

TODO: The discussion around identity and credentials should yield a method to
judge if a new credential is a valid update to an existing one.

## Join the group using an external commit

A request from a client to join a group using an external commit, i.e. without
the help of an existing group member. This operation can be used, for example,
by new clients of a user that already has clients in the group, or by existing
group members that have to recover from state loss.

To retrieve the information necessary to create the external commit, the joiner
has to fetch the external commit information from the DS.

~~~
struct {
  MLSGroupUpdate group_update;
} ExternalJoinRequest;
~~~

**Validation:**

* The MLSGroupUpdate MUST contain a PublicMessage that contains a commit with sender
  type NewMemberCommit.
* The sender of the ExternalJoinRequest MUST be a client that belongs to a user
  that is already in the group.

## Send an application message to a group

A request from a client to fan out an application message to a group. This
operation is meant to send arbitrary data to the rest of the group. Since the
application message is a PrivateMessage, the DS can not verify its content or
authenticate its sender (even though it does authenticate the sender of the
surrounding DSRequest).

~~~
struct {
  MLSMessage application_message;
} SendMessageRequest;
~~~

**Validation:**

* The MLSMessage MUST contain a PrivateMessage with ContentType application.

## Fetch the DS' signature public key

A request from a remote DS to retrieve the signature public key of this DS. The
signature public key can be used by other DSs to verify DSFanoutRequests sent by
the DS. While the DS also uses its signature private key to sign proposals (see
{{proposals-by-reference}}), clients should use the signature key included in
the group's `external_senders` extension to validate those.

The DS responds with a DSResponse of type SignaturePublicKey that contains the
signature public key of this DS.

## Fetch KeyPackages of one or more clients

A request from a client to retrieve the KeyPackage(s) of one or more clients of
this DS. KeyPackages are required to add other clients (and thus other users) to
a group.

~~~
struct {
  ClientID client_identifiers<V>;
} KeyPackagesRequest;
~~~

The DS responds with the KeyPackages of all clients listed in the request.

**Validation:**

* All client identifiers MUST refer to clients native to this DS.
* The DS SHOULD verify that the sender of the request is authorized to retrieve
  the DSKeyPackages of the clients in question. For example, it could check if
  the user of the sending client has a connection with the user of the target
  client(s).

## Fetch connection KeyPackages of one or more clients

A request from a client to retrieve the KeyPackage(s) of all clients of a user
of this DS. KeyPackages obtained via this operation can only be used to add
clients to a connection group.

Connection KeyPackages are available separately from regular KeyPackages, as
they are meant to be accessed by clients of users with which the owning user has
no connection.

~~~
struct {
  UserID user_identifier;
} ConnectionKeyPackagesRequest;
~~~

The DS responds with connection KeyPackages of all clients of the user
corresponding to the identifier in the request.

**Validation:**

* All client identifiers MUST refer to clients native to this DS.
* All clients referred to by the identifiers MUST belong to the same user.

## ReInitialize a group

A request from a client to re-initialize a group with different parameters as
outlined in {{group-lifecycle}}.

~~~
struct {
  MLSGroupUpdate commit;
} ReInitializeGroupRequest;
~~~

**Validation:**

* The MLSGroupUpdate MUST contain a PublicMessage that contains a commit with a
  re-init proposal.
* The GroupID in the re-init proposal MUST point to another group owned by the
  DS, which has a MIMI DS protocol version that is greater or equal than this
  group.

# DSFanoutRequests and DS-to-DS authentication

After the DS has processed an incoming MLSMessage, it prepares a DSFanoutRequest
as described in {{framing-and-processing-overview}}.

To authenticate these messages, an additional layer of DS-to-DS authentication
is required. As defined in {{framing-and-processing-overview}}, DSFanoutRequests
are signed using signing key of the sending DS. The receiving DS can obtain the
corresponding signature public key by sending a DSRequest to the sender
indicated in the DSFanoutRequest.

The request for the signature public key MUST be sent via an HTTPS secured channel, or
otherwise authenticated using a root of trust present on the DS.

TODO: If the transport provides server-to-server authentication this section and
the signature on the DSFanoutRequest can be removed.
TODO: Details on key management, caching etc. We can probably get away with
using very small lifetimes for these keys, as they can be re-fetched cheaply and
essentially at any time.

# Role-based access control in groups

TODO: Access control is beyond the charter. However, to show how easily
implementable it is with MLS, this is section sketches a possible MLS extension
to handle access control in groups that is enforcible and verifiable by both
clients and DS. It is just an example and details can be changed later.

As group operations are handled by MLS PublicMessages, the DS can enforce access
control policies on groups. The privileges of clients in a group are determined
by the group's RBAC GroupContext Extension.

~~~
struct {
  uint32 admins<V>;
} RBACExtension
~~~

The RBACExtension supports two roles: Members and Admins. Since all group
members are Members by default, the extension only lists admins.

Any client the leaf node index of which is listed in the `admins` field of the
RBACExtension in a given group is considered as an Admin role and allowed to
perform the following operations in the context of this group.

* AddUsers, RemoveUsers, DeleteGroup, SetRole

The SetRole operation is an additional operation available to groups that have
an RBACExtension.

~~~
struct {
  MLSGroupUpdate group_update;
} SetRoleRequest
~~~

The `group_update` needs to contain a commit which commits to a single
RBACProposal.

~~~
struct {
  uint32 promoted_members;
  uint32 demoted_members;
} SetRole
~~~

The DS (and all clients) process this proposal by changing the role of the group
members with the leaf indices listed in the `promoted_members` and
`demoted_members` fields and change the `admins` field of the group's
RBACExtension accordingly. For example, if the leaf index admin is listed in the
`changed_members` field of the proposal, it is demoted to Member and removed
from the `admins` field. Similarly, if a Member is listed, it is promoted to
Admin and added to the `admins` field.

# Rate-limiting and spam prevention

All requests (with the exception of the VerificationKeyRequest) can be
explicitly authenticated by the recipient through the verification of a
signature. This means that recipients can follow a rate-limiting strategy of
their choice based on the sender's identity.

For DSRequests, the DS can rate-limit on a per group-level, per-DS level
(reducing the messages from all clients or users belonging to a single DS), or
even based on individual clients or users.

For DSFanoutRequests, rate-limiting or blocking can happen based on the identity
of the sending DS, or it can happen on a per-group basis, where the recipient
only blocks messages from a particular group.

Such rate-limiting can happen by decision of the DS itself, or on the request of
its local users. For example, a user might wish to block connection requests
from one specific other user without blocking connection requests from all other
users of that DS.

# Abusive and illegal content

As all application messages are encrypted, the DS has no way of analyzing their
content for illegal or abusive content. It may make use of a message franking
scheme to allow its users to report such content, although this is beyond the
scope of this document.

Additionally, in the same way as a DS might allow its users to block certain
messages from specific users in the context of spam prevention, it may do the
same based on abusive or illegal content.

# Security considerations

TODO: There is currently no consensus in the MIMI w.r.t. the security goals we
want to reach.

The underlying MLS protocol provides end-to-end encryption and authentication
for all MLSMessages, as well as group state agreement.

## Transport Security

Transport of DSFanoutRequests, as well as their responses MUST use a recipient
authenticated transport. This is to ensure that these messages are mutually
authenticated.

To protect the metadata in all request-response flows, requests and responses
SHOULD be secured using an encrypted transport channel.
