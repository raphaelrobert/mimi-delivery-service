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
 * Privacy-preserving message delivery: The Delivery Service can deliver
   messages to the intended recipients without learning the identity of group
   members, in particular the sender or the recipients of messages. This feature
   is optional and is compatible with all other features, such as assistance
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

The delivery service can operate in one of two modes: A normal mode and a
privacy perserving mode, where the individual operations leak as little metadata
as possible with each operation. The protocol defined in this document allows
for both modes. For simplicity, the main part of this document specifies only
the parts relevant for the normal mode, with the remaining parts specified in
Section XXX (TODO: Link to metadata minimal section).

The main part of this document defines the normal mode with

# Terminology

This document uses the terminology defined in the MLS protocol specification
with the following additions:

TODO: For now and for simplicity, we require that DS' be reachable directly
under their domain. Domain delegation is easily possible and will be defined in
the future.
* DS Domain: Fully qualified domain name as defined in RFC 2181 under which a
  given DS can be reached and which is indicated by the identifiers of their
  users.
TODO: User/client/identifiers definitions are preliminary.
* User: The operator of one or more clients, identified by its
  user identifier.
* User identifier: An identifier that is unique in the scope of its DS and that
  allows resolution to the providers DS domain.
* Client: An MLS client with a unique client identifier.
* Client identifier: An octet string that uniquely identifies the client. A
  client identifier includes the identifier of its user or otherwise allows the
  association betwen the client and its user.
* Connection group: An MLS group consisting of the clients of two users. For
  each pair of users, there can only be one connection group between them. A
  connection group is created when one user requests a connection with another
  user, followed by that user's consent.


# Architecture

The DS consists of three distinct components: A fan-out service (FS), a queueing
service (QS) and an authentication service (AS).

## Fan-out service

As suggested by its name, the main purpose of the FS is to provide endpoints
that allow clients to fan messages out to one of their groups. To this end, the
FS allows clients to create groups and manage group membership. The FS keeps
track of group state by processing the MLS (handshake) messages it fans out.

Since the FS effectively mirrors the (public) group state of clients by
processing MLS messages, it can assist clients in joining new groups by
providing the required information either for Welcome-message or external-commit
based join operations. This feature is crucial, for example, to allow members to
recover from state-loss.

When the FS receives a group message, it fans out the message to the QS of each
group member.

## Queueing service

The role of the QS is on the one hand to store-and-forward messages for its
clients. On the other hand, it allows its clients to store KeyPackages for
retrieval by other clients.

## Authentication service

Before the FS fans out messages or performs group operations, it needs to
authenticate sending clients and potentially verify the association between a
client and its user.

The role of the AS is to provide this capability. More concretely, an owning DS
can contact the AS of a guest DS to obtain the key material required to
authenticate guest clients and users.

The AS also allows remote clients to discover clients of the local DS and obtain
KeyPackages for use in connection groups. These KeyPackages differ from regular
KeyPackages provisioned via the QS in that they are not meant to be used in
regular groups.


# Protocol overview

MLS is fundamentally a client-to-client protocol, where the DS only plays a
supporting role. As this protocol leverages MLS for a variety of functionalities
such as authentication and agreement, it is essentially a client-to-server
protocol, although messages from guest clients can be proxied through their own
DS, thus making it a server-to-server protocol in the federated context.


## Flow

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

Depending on the client's request, the Delivery Service might send a message to
the QS of each guest DS. This happens whenever a message needs to be fanned out
to all other members of a group. The guest DS can respond, for example, that one
of the target clients has been deleted.

~~~ aasvg

Client           Owning Delivery Service           Guest Delivery Service
|                |                                 |
| DSRequest      |                                 |
+--------------->|                                 |
|                |                                 |
| DSResponse     |                                 |
|<---------------+                                 |
|                | DsFanoutRequest                 |
|                +-------------------------------->|
|                |                                 |
|                | DsFanoutResponse                |
|                |<--------------------------------+
|                |                                 |
~~~
{: title="Client/Delivery Service communication with fanout" }

In the federated case, messages to a guest Delivery Service can be proxied
via the sender's own Delivery Service.

## Supported operations

The MIMI DS protocol supports a variety of operations that allow clients to
manage group membership, join groups, or send messages.

TODO: Some of these operations are decidedly client-server and not particularly
relevant in the federated setting (e.g. user registration, upload of
KeyPackages, etc.). Do we still want them here?

FS operations:

 * Group creation/deletion
 * Join a group from a Welcome message as a new member
 * Join a group through an External Commit message as a new member
   or client of an existing member
 * Adding and removing users to/from a group
 * Adding and removing clients to/from a member of the group
 * Client updates (MLS leaf updates)
 * Resync a client with a group in case of state loss
 * Sending application messages

QS operations:

 * Upload of KeyPackages
 * Download of KeyPackages
 * Enqueueing of a message fanned out from a local or a remote FS
 * Retrieval of previously enqueued messages

AS operations:

 * User registration/deletion
 * Client addition/removal
 * Discovery of users, their clients and the associated authentication key
   material
 * Upload of connection-specific KeyPackages
 * Download of connection-specific KeyPackages

## Client removals induced by the Delivery Service

 The Delivery Service can remove clients from a group by issuing remove
 proposals in the following cases:

 * A user has removed a client from its account
 * A user has been deleted
 * The client is removed from the group because it has been inactive for too
   long

## Serialization format

This document uses the presentation language and encoding format defined in RFC
8446 (TODO: Link) with the extensions defined in the MLS protocol specification.
The bytes strings resulting from serialization in this format are unambiguous
and require not further canonicalization.


## Framing and processing overview

TODO: We're mixing framing and message processing slightly here, which makes
sense from a presentational standpoint. Later, we go into more detail w.r.t. to
processing and especially authentication.

All DSRequest messages consist of a MIMI DS specific protocol wrapper called
DSMessage. DSMessage contains the MIMI DS protocol version, a body with
operation-specific data, as well as authentication information.

~~~
enum {
  ds_create_group(0),
  ds_delete_group(1),
  ...
} DSRequestType;

struct {
  DSRequestType request_type;
  select (DSRequestBody.request_type) {
    case fs_create_group:
      CreateGroupRequest create_group_request;
    case fs_delete_group:
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
partially here. A more complete definition with all operations relevant for the
normal DS operating mode can be found in Section XXX (TODO: Link to Operations
section), while the full definition including the operations relevant for the
privacy-preserving operating mode and can be found in Section XXX (TODO: Link).

The `authentication_data` field of a DSRequest depends on the request type and
contains the data necessary for the DS to authenticate the request.

~~~
enum {
  Anonymous,
  ClientSignature,
  ...
} DSAuthType;

struct {
  DSAuthType auth_type;
  select (DSAuthData.auth_type) {
    case Anonymous:
      struct {};
    case ClientSignature:
      uint32 sender_index;
      opaque signature<0..255>;
    ...
  }
} DSAuthData;
~~~

For the normal DS operating mode, Anonymous and ClientSignature are the only
relevant authentication types. The complete specification of DSAuthType and
DSAuthBody can be found in Section XXX (TODO: Link to metadata minimal section).

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

Note that all group operations additionally contain an MLS message the content
of which mirrors the request type, e.g., an AddUsers request wraps an MLS commit
that in turn contains the Add proposals for the users' clients. In that case,
the DS uses the GroupID inside the MLS message to determine which group the
request refers to and verifies the MLS message in the same way an MLS client
would (including the signature).

Depending on the nature of the request, clients can also include data in the AAD
field of the MLS message, where it can be read and authenticated by both DS and
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
  VerifyingKey,
  KeyPackages,
  ConnectionKeyPackages,
}

enum DSResponseBody {
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
    case VerifyingKey:
      SignaturePublicKey verifying_key;
    case KeyPackages:
      KeyPackage key_packages<V>;
    case ConnectionKeyPackages:
      KeyPackage key_packages<V>;
  }
}

enum {
  TODO: Operation specific errors.
} DSError
~~~

After sending the response, and depending on the operation the DS might fan out
messages to one or more guest DS'.

To that end, it wraps the MLS message to be fanned out into a DSFanoutRequest.
In addition to the MLS message, the DSFanoutRequest contains the protocol
version, the FQDN of the sending DS and the identifiers of all clients on DS
that the message is meant to be fanned out to.

If the message needs to be fanned out to more than one guest DS, the DS prepares
different messages for each destination DS with each message containing only the
DSFanoutRecipients of that DS.

~~~
enum {
  ClientIdentifier,
  ...
} DSFanoutRecipientType;

struct {
  DSRecipientType recipient_type;
  select (DSRecipient.recipient_type) {
    case ClientIdentifier:
      opaque client_id<0..255>;
    ...
  }
} DSFanoutRecipient;

struct {
  DSProtocolVersion protocol_version;
  FQDN sender;
  DSFanoutRecipient recipients<V>;
  MLSMessage mls_message;
  // Signature over the above fields
  opaque signature<0..255>;
} DSFanoutRequest
~~~

The DS receing a DSFanoutRequest can then store-and-forward the contained MLS
message to the clients indicated in the `recipients` field.

TODO: We need to figure out how we want to react to DSFanoutRequests. In the
normal operating mode, we don't necessarily have to use these reactions to
propagate client deletions. Instead, servers could just keep track of which
remote DS' individual clients have groups on and send dedicated requests to them
to have the client deleted. We may even want to add them as external senders.

TODO: How to best serialize an FQDN into a byte string? We could probably just
use UTF-8 for now and restrict characters to ascii as necessary.

TODO: If we really wanted to, we could unify the DSRequest and DSFanoutRequest
structs. For now, splitting it up according to the protocol flow is fine.

For presentational reasons, we only define DSRecipientType and the corresponding
`case` statement in DSRecipient partially here. The rest of the definition is
relevant only for the privacy-preserving operating mode and can be found in
Section XXX (TODO: Link).

The receiving DS verifies the signature using the sending DS' public signature key.

## Connection establishment flow


A user can establish a connection to another user by creating a connection
group, fetching connection KeyPackages of the target user's clients and inviting
that user to the connection group. The receiving user can process the Welcome
and figure out from the group state who the sender is. Additional information
can either be attached to the group via an extension, or via MLS application
messages within that group.

TODO: This is a sketch for a simple connection establishment flow that allows
the recipient to identify and authenticate the sender and that establishes an
initial communication channel between the two users. More functionality can be
added via additional messages or MLS extensions.

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
  ds_resync_client(9),
  ds_send_message(10),
  ds_verifying_key(11),
  ds_key_packages(12),
  ds_connection_key_packages(13),
  ...
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
    case ds_resync_client:
      ResyncClientRequest resync_client_request;
    case ds_send_message:
      SendMessageRequest send_message_request;
    case ds_verifying_key:
      struct {};
    case ds_key_packages:
      KeyPackagesRequest key_packages_request;
    case ds_connection_key_packages:
      ConnectionKeyPackagesRequest connection_key_packages_request;
    ...
  }
} DSRequestBody;
~~~

A number of these operations require either slightly different or additional
parameters in the privacy preserving operational mode. If the parameter is
different, it will be defined as an enum with one variant for each mode. If
additional parameters are required, a field with an optional type is included
that contains a type prefixed with "PP" to indicate that it is relevant only for
the privacy preserving mode.

Additional variants specific to the privacy preserving mode, as well as the
corresponding rest of the `case` statement can be found in Section XXX (TODO:
add link).

## MLSMessages and GroupInfos

To verify and deliver messages, authenticate clients as members of a group and
to assist clients that want to join a group, the DS keeps track of the state of
each group it owns. More specifically, it keeps track of the group's ratchet
tree, the group's GroupContext and other information required to produce a valid
GroupInfo for the current group epoch. It does this by processing incoming MLS
messages in the same way a member of that group would, except of course that the
DS doesn't hold any private key material.

While MLSMessages are sufficient to keep track of most of the group information,
it is not quite enough to create a GroupInfo. To allow the DS to provide a valid
GroupInfo to externally joining clients, it additionally requires clients to
provide the remaining required information. Concretely, it requires clients to
upload the Signature and the GroupInfo extensions. Clients need to send this
information whenever they send an MLSMessage that contains a commit.

~~~
struct {
  Extension group_info_extensions<V>;
  opaque Signature<V>;
} PartialGroupInfo
~~~

The combination of a commit and a partial group info is called an
AssistedMLSMessage.

~~~
struct {
  MLSMessage commit;
  PartialGroupInfo partial_group_info;
} AssistedMLSMessage
~~~

Whenever the DS receives an AssistedMLSMessage, it must verify that the
MLSMessage contains a PublicMessage with a commit and that commit and partial
group info are valid relative to the existing group state according to the MLS
specification.


TODO: How do we want to structure the responses? We will have very few differen
"OK" type responses, but since every response will likeyly have its own error
messages, we might want to define a request-specific error struct. For now, I'll
skip defining error responses completely.
TODO: For now, we only specify the federation-relevant parts, i.e. no queue
retrieval, etc.
TODO: For now there is no distinct endpoint to obtain AS-specific authentication
material, i.e. the authentication material the DS uses to sign the signature
keys of its clients/users.
TODO: Right now, we only have two auth methods: No authentication and
group-specific authentication. We will likely have to define additional auth
methods.
TODO: Right now, we always send a full GroupInfo, where a partial GroupInfo
containing only the extensions and the signature would be enough. We should
properly define a partial group info, describe what it does and integrate it
into other struct definitions. In the same section, we should also describe how
the DS tracks the group state and processes the MLS messages.
TODO: Besides the resync endpoint, we don't have a specific endpoint for
external joins. Do we want one?
TODO: Add section on how "extra" proposals are handled, e.g., self-remove
proposals, or proposals sent by the DS.
TODO: Note that the DS with its verifying key needs to be set as a valid
external sender in all groups.

## Request group id

TODO: This is not a super relevant operation for the federated case. Do we want
this in here?

Clients can use this operation to request a group id. This group ID can
subsequently used to create a group on this DS.

After receiving this request, the DS generates a unique group id an responds
with a DSResponse struct containing the group id.

## Create group

TODO: This is not super relevant operation for the federated case. Do we want
this in here?

A request from the client to create a new group on the Delivery Service.

The client sends the following CreateGroupRequest to the Delivery Service:

~~~
struct {
  opaque group_id<V>;
  LeafNode LeafNode;
  MLSMessage group_info;
  optional<PPCreateGroupData>
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
 * The LeafNode is valid, according to (I-D.ietf-mls-protocol) 7.3. Leaf Node
   validation.
 * The GroupInfo is valid, according to (I-D.ietf-mls-protocol) 12.4.3. Adding
   Members to the Group.

## Delete group

A request from the client to delete a group from the Delivery Service.

~~~
struct {
  AssistedMLSMessage assisted_message;
} DeleteGroupRequest;
~~~

### Validation

The Delivery Service validates the request as follows:

 * The AssistedMLSMessage must contain a PublicMessage with a commit that contains
   remove proposals for every member of the group except the committer.


## Add users to a group

A request from a client to add one or more users to a group.

~~~
struct {
  AssistedMLSMessage assisted_message;
  MLSMessage welcomes<V>;
  optional<PPAddUsersData> pp_data;
} AddUsersRequest;
~~~

### Validation

TODO: I have left out KeyPackageBatches for now and we don't have user-level
identities, so there is no way to check if all clients of a given user are
added.
* The AssistedMLSMessage in the `commit` field must contain a PublicMessage with a
  commit that contains only Add proposals.
* Add proposals must not contain clients of existing group members.


## Remove users from a group

A request from a client to remove one or more users from a group.

~~~
struct {
  AssistedMLSMessage assisted_message;
} RemoveUsersRequest;
~~~

### Validation

* The AssistedMLSMessage must contain a PublicMessage with a commit that contains only
  remove proposals.
* If the commit contains a remove proposal that targets one client of a user in the
  group, other remove proposals in the commit must target the other clients of
  that user.

## Add clients to a group

A request from a client to add one or more clients of the same user.

~~~
struct {
  AssistedMLSMessage assisted_message;
  MLSMessage welcomes<V>;
  optional<PPAddClientsData> pp_data;
} AddClientsRequest;
~~~

### Validation

* The AssistedMLSMessage must contain a PublicMessage with a commit that contains only
  add proposals.
* All Add proposals must contain clients of the same user as an existing group
  member.

## Remove clients from a group

A request from a client to remove one or more other clients of the same user
from a group.

~~~
struct {
  AssistedMLSMessage assisted_message;
} RemoveClientsRequest;
~~~

### Validation

* The AssistedMLSMessage must contain a PublicMessage with a commit that contains only
  remove proposals.
* All remove proposals must target clients of the same user as the sending
  client.

## Self remove a client from a group

A request from a client to remove itself from the group. If it's the last client
of a user, this effectively removes the user from the group.

~~~
struct {
  MLSMessage proposal;
} SelfRemoveClientRequest;
~~~

TODO: Add discussion regarding the difficulty of self-removals in MLS.

### Validation

* The MLSMessage must contain a PublicMessage that contains a single
  remove proposal.
* The remove proposal must target the sending client.

## Update a client in a group

A request from a client to update its own leaf in an MLS group.

~~~
struct {
  AssistedMLSMessage assisted_message;
  optional<PPUpdateClientData> pp_data;
} UpdateClientRequest;
~~~

### Validation

* The AssistedMLSMessage must contain a PublicMessage that contains a commit with an
  UpdatePath, but without other proposals.

## Resync a client in a group

A request from a client to recover from state loss by re-adding themselves to a
group.

~~~
struct {
  AssistedMLSMessage assisted_message;
  optional<PPResyncClientData> pp_data;
} ResyncClientRequest;
~~~

### Validation

* The AssistedMLSMessage must contain a PublicMessage that contains a commit with sender
  type NewMemberCommit and with a remove proposal.

## Send an application message to a group

A request from a client to fan out an application message to a group.

~~~
struct {
  MLSMessage application_message;
} SendMessageRequest;
~~~

### Validation

* The MLSMessage must contain a PrivateMessage with ContentType application.

## Fetch the DS' verifying key

TODO: Terminology: signature public key or verifying key? We use
SignaturePublicKey in the MLS spec, but I much prefer verifying key at this
point. Once we decide, we should stick with the same throughout this document.

A request from a remote DS to retrieve the verifying key of this DS.

The DS responds with a DSResponse of type VerifyingKey that contains the
verifying key of this DS.

## Fetch KeyPackages of one or more clients

A request from a client to retrieve the KeyPackage(s) of one or more clients of
this DS.

~~~
struct {
  ClientID client_identifiers<V>;
} KeyPackagesRequest;
~~~

The DS responds with the KeyPackages of all clients listed in the request.

### Validation

TODO: Should all clients have to belong to the same user?
* All client identifiers must refer to clients native to this DS.

## Fetch connection KeyPackages of one or more clients

A request from a client to retrieve the KeyPackage(s) of all clients of a user
of this DS.

~~~
struct {
  UserID user_identifier;
} ConnectionKeyPackagesRequest;
~~~

The DS responds with connection KeyPackages of all clients of the user
corresponding to the identifier in the request.

### Validation

* All client identifiers must refer to clients native to this DS.
* All clients referred to by the identifiers must belong to the same user.

# DSFanoutRequests and DS-to-DS authentication

After the DS has processed an incoming MLSMessage, it prepars a DSFanoutRequest
as described in Section XXX (TODO: Link).

To authenticate these messages, an additional layer of DS-to-DS authentication
is required. As defined in Section XXX (TODO: Link), DSFanoutRequests are signed
using the sending DS' signing key. The receiving DS can obtain the corresponding
verifying key by sending a DSRequest to the sender indicated in the
DSFanoutRequest.

The request for the verifying key MUST be sent via an HTTPS secured channel, or
otherwise authenticated using a root of trust present on the DS.
TODO: I'm being a bit vague here since we don't have a fixed transport protocol
yet.

The receiving DS may cache the verifying key for X amount of time.

TODO: Details on key management. We can probably get away with using very small
lifetimes for these keys, as they can be re-fetched cheaply and essentially at
any time.

# Role-based access control in groups

TODO: This is section sketches a possible MLS extension to handle access control
in groups that is enforcible and verifiable by both clients and DS. It is just
an example and details can be changed later.

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

TODO: It's a bit weird to specify this operation here as opposed to the
Operations section, since it also needs its own entry in the RequestType enum.
We'll move it there once we decide that this extension makes sense for now.
Also: Do all groups have to have this RBACExtension? It's a little overhead, but
if groups want, they can just set all group members to Admin.

The SetRole operation is an additional operation available to groups that have
an RBACExtension.

~~~
struct {
  AssistedMLSMessage assisted_message;
} SetRoleRequest
~~~

The assisted_message needs to contain a commit which commits to a single
RBACProposal.

~~~
struct {
  uint32 changed_members;
} SetRole
~~~

The DS (and all clients) process this proposal by toggling the role of the group
members with the leaf indices listed in the `changed_members` field and change
the `admins` field of the group's RBACExtension accordingly. For example, if the
leaf index admin is listed in the `changed_members` field of the proposal, it is
demoted to Member and removed from the `admins` field. Similarly, if a Member is
listed, it is promoted to Admin and added to the `admins` field.

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
content for illegal or abusive content. However, it may make use of a message
franking scheme to allow its users to report such content.

Additionally, in the same way as a DS might allow its users to block certain
messages from specific users based on spam, it may do the same based on abusive
or illegal content.


# Privacy preserving message delivery

If the desired mode of operation is for the Delivery Service to learn as little
as possible about the groups it owns and their individual members, the protocol
can be extended to protect the group state on the Delivery Service. This can
happen through two complementary mechanisms:

* Unlinking member credentials from credentials issued by the Authentication
  Service, providing pseudonymity at the Delivery Service level
* Encrypting the group state with a key that is only known to the group
  members, providing encryption at rest

In practice, the requests from clients to the Delivery Server are extended with
additional parameters, such as decryption keys for the group state and
additional pseudonymous user-level authentication.

The only major change to the protocol that is required in the privacy preserving
mode of operation is moving the connection establishment flow from a Welcome
based one to one based on external commits. While this change prevents the
initiator from immediately sending messages after creating the connection group,
it is necessary, because the use of KeyPackages allows the DS to track which
group is used as the connection group.

TODO: Add section on security considerations.