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
 * Assistance for new joiners of a group: The Delivery Service can keep and
   update state such as the public ratchet tree, group extensions, etc.
 * Message validation: The Delivery Service can inspect and validate handshake
   messages and reject malformed, invalid or malicious messages.
 * Built-in authentication of group members and policy enforcement: The Delivery
   Service can authenticate group members and reject messages from non-members.
   Additionally, the Delivery Service can be linked to the Authentication
   Service to enforce more policies and validation rules.
 * Scalability: The protocol makes no assumption about whether the Delivery Service
   runs as a single instance or as a cluster of instances.
 * Network fluid: While the Delivery Service would typically run as a
   server-side component, the only requirement is that it is accessible by all
   clients.
 * Transport agnostic: Messages between clients and the Delivery Service can be
   sent via an arbitrary transport protocol. Additionally, in the federated
   case, client messages to a guest Delivery Service can be forwarded by a
   client's local instance of this service.

TODO: Make use of MUST/SHOULD, etc. throughout.

# Terminology

This document uses the terminology defined in {{!RFC9420}} with the following
additions:

* DS Domain: Fully qualified domain name as defined in {{!RFC2181}} that
  represents the DS. This does not necessarily have to be the domain under which
  the DS is reachable on the network. Discovering that domain is the
  responsibility of the transport protocol over which the MIMI DS protocol runs.
TODO: client identifiers definitions are preliminary.
* Client: An MLS client with a unique client identifier.
* Client identifier: An octet string that uniquely identifies the client and
  that maps to the DS domain of the client's DS.

# Interfaces

The MIMI DS protocol inherits the proposal-commit logic from MLS. Any party that
is either a group member, or that as added as an external sender can propose
changes to the group such as the addition, removal, or update of clients. Group
members can then change the group state in a commit operation. A commit MUST
include all valid previously received proposals, as well as any valid proposals
that the committing client wants to add to the commit.

The MIMI DS tracks the state of individual MLS groups on a hub by storing
proposals and processing commits.

Any party (including group members and external senders) can send requests for
the current group information (an MLS GroupInfo, required for clients to join
the group without the help of a group member) or request key material (MLS
KeyPackages, required by group members to add new group members) of the DS'
clients.

Group members and external senders can send send proposals to the group. Valid
proposals include any proposals in {{!RFC9420}}:

* Add proposals (to add new group members) 
* Remove proposals (to remove existing group members) 
* Update proposals (to update a group members's key material)
* PSK proposals (to inject additional key material into the group)
* Re-Init proposals (to re-initialize the group, e.g. with a new ciphersuite)
* Group Context Extensions (to modify data in GroupContext Extensions, e.g. to
  add/remove/update external senders)

TODO: We might want to allow non-group members to propose adding themselves
(NewMemberProposal).

Additional proposals may be added via MLS' extension mechanism.

Group members can send commits (which can include any of the proposals listed
above), as well as application messages.

Clients that are not group members can send an (external) commit to add
themselves to a group.

Group members can send an (external) commit to re-join the group (e.g. if they
have previously lost state, or their group state was corrupted).

The DS will verify all proposals, commits and application messages as described
in {{!RFC9420}} and fan them out to the rest of the group.

# Architecture and protocol overview

The MIMI DS protocol allows interoperability between a hub which hosts a
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
the hub to track the MLS group state in the same way as a client would,
enabling it to keep an up-to-date and fully authenticated list of members, as
well as provide the full MLS group state to joining group members, even for
those joining via external commit. In addition, the DS can verify messages and
enforce access control policies on group operations.

## Client to server and server to server protocol {#c2s}

MLS being a protocol for end-to-end encryption, the MLS protocol messages that
many of the MIMI DS protocol messages have to originate from the clients rather
than the interoperating delivery services.

The MIMI DS protocol consists of two parts: A client-to-server part that allows
guest clients to interact with the hub of one of their groups and a
server-to-server protocol that allows a hub to fan out messages to guest
DSs, which can subsequently store and forward messages to their respective
clients.

Note that the client-to-server part of the protocol can optionally be proxied
via the guest DS of the sending client.

## Transport for the MIMI DS protocol

The MIMI DS protocol requires a transport protocol that provides confidentiality
of messages and that allows the discovery of a DS based on the DS domain in a
client identifier.

The client-to-server part of the MIMI DS protocol provide sender authentication.
Recipient authentication, as well as mutual server-to-server authentication is
left to the MIMI transport protocol.

## Flow

~~~aasvg
+-------------+              +--------------+
|             +------------->+              |
| Sending     | (proprietary | Guest DS     |
| Client      |   protocol)  |              |
|             +<-------------+              |
+-------------+              +--+--------+--+
                                |        ^
                      DSRequest |        | DSResponse
                                v        |
                             +--+--------+--+
                             |              |
                             | Hub          |
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
{: #full-sending-flow title="Architecture overview" }

{{full-sending-flow}} shows an example protocol flow, where a client sends a
request to its own guest DS, which in turn makes a request to the hub. The
hub then fans out a message to another guest DS.

Both the message sending and the fanout parts of the protocol are designed in a
request/response pattern. In the first protocol part, the client sends a
DSRequest message to the Delivery Service and the Delivery Service responds with
a DSResponse message. This pattern can easily be used over e.g. RESTful APIs.

~~~aasvg
Client           Hub
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
Client           Hub                               Guest Delivery Service
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

## Enqueue authorization

TODO: This section sketches an authorization mechanism based on a KeyPackage
extension. That extension would have to be defined in the context of the MLS WG.

Each KeyPackage that a client publishes also carries a FanoutAuthToken inside
a FanoutAuth KeyPackage extension.

~~~ tls
struct {
  opaque token<V>;
} FanoutAuthToken
~~~

Whenever a client is added to a group (or when a new group is created), the DS
checks that the KeyPackages of all joiners contain a FanoutAuth extension and
stores the contained token alongside the group state.

The FanoutAuthToken is included in DSFanoutRequests and allows the receiving
DS to check whether the sender is authorized to enqueue messages for the
recipient.

Clients can change their FanoutAuthToken by sending a new token with an
update operation.

TODO: Details on the cryptographic scheme underlying the token.

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
group can be re-initialized as described in Section 11.2. of {{!RFC9420}} to
create a new group with a new set of parameters.

The MIMI DS protocol supports re-initializations of groups using the
corresponding ReInitialization operation under the condition that all MLS
parameters are compatible with the MIMI DS protocol version.

If a group is no longer used, it can be deleted either by a client or the DS
itself.

TODO: Each MIMI DS protocol version should probably fix a set of ciphersuites,
MLS protocol versions and maybe even extensions it supports. New ones can be
added with protocol version upgrades.

# Security properties overview

The MIMI DS protocol provides a number of security guarantees.

## MLS-based security properties

The MLS protocol underlying the MIMI DS protocol provides a number of security
guarantees that primarily affect end-to-end communication between clients such
as authentication and confidentiality with forward- as well as post-compromise
security (although the latter only holds for application messages). While the
MIMI DS protocol acts as a layer around the MLS protocol it inherits some of
MLSs security guarantees.

More concretely, the DS can verify client signatures on MLS messages and thus
ensure that end-to-end authentication holds. Since the DS uses MLS messages to
track the group state (including group membership), it is guaranteed to have the
same view of that state as the group members.

# Framing and processing overview

## Client to server requests

All client to server requests consist of a MIMI DS specific protocol wrapper
called DSRequst. DSRequest contains the MIMI DS protocol version, a body with
operation-specific data, as well as authentication information.

~~~ tls
enum {
  ds_delete_group(0),
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

~~~ tls
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

~~~ tls
struct {
  DSProtocolVersion protocol_version;
  DSRequestBody request_body;
  u32 sender_index
} ClientSignatureTBS
~~~

Note that all group operations additionally contain an MLSMessage the content of
which mirrors the request type, e.g., an AddClients request wraps an MLS commit
that in turn contains the Add proposals for the clients to be added. In that
case, the DS uses the GroupID inside the MLSMessage to determine which group the
request refers to and verifies the MLSMessage in the same way an MLS client
would (including the signature).

Depending on the nature of the request, clients can also include data in the AAD
field of the MLSMessage, where it can be read and authenticated by both DS and
all other group members.

After performing the desired operation using the data in DSRequestBody the DS
responds to the client (or the proxying guest DS) with a DSResponse.

~~~ tls

struct {
  DSProtocolVersion protocol_version;
  DSResponseBody response_body;
} DSResponse

enum DSResponseType {
  Ok,
  Error,
  WelcomeInfo,
  ExternalCommitInfo
  KeyPackages,
}

struct DSResponseBody {
  DSResponseType response_type;
  select (DSResponseBody.response_type) {
    case Ok:
      struct {};
    case Error:
      DSError error;
    case WelcomeInfo:
      optional<Node> ratchet_tree<V>;
    case ExternalCommit:
      MLSMessage: group_info;
      optional<Node> ratchet_tree<V>;
    case KeyPackages:
      KeyPackage key_packages<V>;
  }
}

struct {
  TODO: Operation specific errors.
} DSError
~~~

## Server to server requests

After sending the response, and depending on the operation the DS might fan out
messages to one or more guest DSs.

To that end, it wraps the MLSMessage to be fanned out into a DSFanoutRequest. In
addition to the MLSMessage, the DSFanoutRequest contains the protocol version, a
list of client ids representing the clients for which the payload is meant, a
FanoutAuthToken (see {{enqueue-authorization}}) for each of the clients, as well
as the payload to be fanned out.

~~~ tls
struct {
  DSProtocolVersion protocol_version;
  opaque recipient_ids<V>;
  FanoutAuthToken fanout_auth_tokens<V>;
  MLSMessage mls_message;
} DSFanoutRequest
~~~

The receiving DS first verifies the signature using the sending DS' public
signature key and then further validates the message by performing the following
checks:

* That the protocol version is compatible with its configuration
* That the `recipient_ids` are all clients of this DS
* The the `fanout_auth_tokens` are all valid tokens for the individual
  recipients

The recieving DS can then store and forward the contained MLS message to the
clients indicated in the `recipient_ids` field and send a DSFanoutResponse.


~~~ tls
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

# DS assisted joining

To verify and deliver messages, authenticate clients as members of a group and
to assist clients that want to join a group, the DS keeps track of the state of
each group for which it is the hub. More specifically, it keeps track of the
group's ratchet tree, the group's GroupContext and other information required to
produce a valid GroupInfo for the current group epoch. It does this by
processing incoming MLS messages in the same way a member of that group would,
except of course that the DS doesn't hold any private key material.

While MLS messages are sufficient to keep track of most of the group information,
it is not quite enough to create a GroupInfo. To allow the DS to provide a valid
GroupInfo to externally joining clients, it additionally requires clients to
provide the remaining required information. Concretely, it requires clients to
upload the Signature and the GroupInfo extensions. Clients need to send this
information whenever they send an MLS message (i.e. an MLSMessage struct) that
contains a commit.

~~~ tls
struct {
  Extension group_info_extensions<V>;
  opaque Signature<V>;
} PartialGroupInfo
~~~

The combination of a commit and a partial group info is called an
MLSGroupUpdate.

~~~ tls
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

# Proposals and DS-initiated operations {#proposals-by-reference}

MLS relies on a proposal-commit logic, where the proposals encode the specific
action the sending client intends to take and the commit then performs the
actions of a set of commits.

The advantage of this approach is that the sender of the proposal does not have
to be the committer, which allows, for example, the DS to propose the removal of
a client, or a client to propose that it be removed from the group. Note that
the latter example is the only way that a client can remove itself (i.e. leave)
from a group.

Such proposals, where the original sender differs from the sender of the commit
are called "proposal by reference", or "proposal by value" if the proposal is
sent by the committer as part of the commit itself.

The following sections detail operations that can be performed by clients, so
each operation that entails a change to the group state (with the exception of
the self remove operation) require the sender to perform a commit, where the
semantics of the operation are reflected as proposals by value. For example, the
commit in the AddClientsRequest must only contain Add proposals.

If a client receives a valid standalone proposal, it MUST store it and verify
that the next commit includes said proposal.

TODO: Be more specific about proposal validation. Also, we might want to allow
the server to rescind a proposal.

Whenever a client sends a commit as part of an operation, it MUST include all
stored proposals by reference, such as server-initiated Remove proposals, or
proposals sent as part of a self-remove operation.

TODO: Proposals by reference pose a problem in the context of external commits,
as, even if the external committer had access to all proposals in an epoch, it
wouldn't be able to verify them, thus potentially leading to an invalid external
commit. A solution could be introduced either as part of the MIMI DS protocol,
or as an MLS extension. The latter would be preferable, as other users of MLS
are likely going to encounter the same problem.

To allow the DS to send proposals, all groups MUST contain an `external_senders`
extension as defined in Section 12.1.8.1. of {{!RFC9420}} that includes the DS'
credential and its signature public key.

TODO: We might also want to mandate that the group includes credentials of guest
DSs involved in the group. However, for them to be able to send proposals, we'd
need an additional operation/endpoint that the DS exposes.

TODO: Details of the DS credential, distribution, etc. A BasicCredential with
the FQDN of the DS would probably be sufficient.

The DS can simply create such proposals itself based on the group information
and distribute it to all group members via regular DSFanoutRequests.

# Client-initiated operations {#operations}

The DS supports a number of operations, each of which is represented by a
variant of the DSRequestType enum and has its own request body.

~~~ tls
enum {
  ds_delete_group,
  ds_proposals,
  ds_commits,
  ds_send_message,
  ds_key_packages,
  ds_group_info,
} DSRequestType;

struct {
  DSRequestType request_type;
  select (DSRequestBody.request_type) {
    case ds_delete_group:
      DeleteGroupRequest delete_group_request;
    case ds_proposals:
      ProposalRequest proposal_request;
    case ds_commits:
      CommitRequest commit_request;
    case ds_send_message:
      SendMessageRequest send_message_request;
    case ds_key_packages:
      KeyPackagesRequest key_packages_request;
    case ds_key_packages:
      GroupInfoRequest group_info_request;
  }
} DSRequestBody;
~~~

## Delete group

A request from the client to delete a group from the Delivery Service. This
operation allows clients to delete a group from the DS. Clients can of course
keep a copy of the group state locally for archival purposes.

~~~ tls
struct {
  MLSGroupUpdate group_update;
} DeleteGroupRequest;
~~~

**Validation:**

The Delivery Service validates the request as follows:

 * The MLSGroupUpdate MUST contain a PublicMessage with a commit that contains
   Remove proposals for every member of the group except the committer.

## Propose group state changes

A request from a client or an external sender to propose one ore more changes to
the group state. Each change is encoded as an MLS proposal.

~~~ tls
struct {
  MLSMessage proposals<V>;
  MLSMessage welcome_messages<V>;
} AddClientsRequest;
~~~

**Validation:**

* The MLSMessages MUST contain PublicMessages with proposals.
* All proposals MUST be valid according to {{!RFC9420}}.

## Commit group state changes

A request from a client to change the group state via one or more commits as
proposed by previously sent proposals, or proposals included in the commits.
Note that this operation cannot be used by a group member to remove itself from
the group. Group members can only propose their own removal and wait for another
group member to commit the change.

Clients that are not yet group members can use a commit to add themselves to a
group.

Group members can re-add themselves to a group via a commit (e.g. in case of
lost group state).

~~~ tls
struct {
  MLSGroupUpdate group_updates<V>;
} RemoveClientsRequest;
~~~

**Validation:**

* The MLSGroupUpdates MUST contain PublicMessages that contain commits.

## Send application messages to a group

A request from a client to fan out one or more application messages to a group.
This operation is meant to send arbitrary data to the rest of the group. Since
the application message is a PrivateMessage, the DS can not verify its content
or authenticate its sender (even though it does authenticate the sender of the
surrounding DSRequest).

~~~
struct {
  MLSMessage application_message<V>;
} SendMessageRequest;
~~~

**Validation:**

* The MLSMessages MUST contain PrivateMessages with ContentType application.

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
  the DSKeyPackages of the clients in question.

## Fetch group information

A request from a client to retrieve the group's GroupInfo. KeyPackages are
required for clients to add themselves to a group via a commit and for group
members to re-add themselves to a group.

~~~
struct {} GroupInfoRequest;
~~~

The DS responds with the group's current GroupInfo.

**Validation:**

* The DS SHOULD verify that the sender of the request is authorized to retrieve
  the group's GroupInfo.

# DSFanoutRequests

After the DS has processed an incoming MLSMessage, it prepares a DSFanoutRequest
as described in {{framing-and-processing-overview}}.

For DS to DS communication, the MIMI DS protocol relies on the underlying
transport protocol to provide mutual authentication.

# Rate-limiting and spam prevention

All requests (with the exception of the VerificationKeyRequest) can be
explicitly authenticated by the recipient through the verification of a
signature. This means that recipients can follow a rate-limiting strategy of
their choice based on the sender's identity.

For DSRequests, the DS can rate-limit on a per group-level, per-DS level
(reducing the messages from all clients belonging to a single DS), or even based
on individual clients.

For DSFanoutRequests, rate-limiting or blocking can happen based on the identity
of the sending DS, or it can happen on a per-group basis, where the recipient
only blocks messages from a particular group.

Such rate-limiting can happen by decision of the DS itself, or on the request of
its local clients. For example, a client might wish to block connection requests
from one specific other client without blocking connection requests from all other
clients of that DS.

# Abusive and illegal content

As all application messages are encrypted, the DS has no way of analyzing their
content for illegal or abusive content. It may make use of a message franking
scheme to allow its clients to report such content, although this is beyond the
scope of this document.

Additionally, in the same way as a DS might allow its clients to block certain
messages from specific clients in the context of spam prevention, it may do the
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
