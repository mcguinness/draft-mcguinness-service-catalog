---
title: "Per-User Service Catalog for Agents"
abbrev: "Service Catalog"
docName: "draft-mcguinness-service-catalog-latest"
category: "std"
# workgroup: "Web Authorization Protocol"
# area: "Security"
ipr: "trust200902"
keyword:
  - "OAuth 2.0"
  - "Discovery"
  - "Agents"
  - "Catalog"
  - "MCP"
  - "Authorization"
venue:
#  group: "Web Authorization Protocol"
#  type: "Working Group"
#  mail: "oauth@ietf.org"
#  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  RFC3339:
  RFC6749:
  RFC6750:
  RFC8259:
  RFC8288:
  RFC8414:
  RFC8615:
  RFC8707:
  RFC9111:
  RFC9396:
  RFC9728:

informative:
  RFC7033:
  RFC7591:
  RFC8693:
  RFC9126:
  RFC9470:
  RFC9635:
  RAR-METADATA:
    title: "Metadata for OAuth 2.0 Rich Authorization Requests"
    target: https://datatracker.ietf.org/doc/draft-zehavi-oauth-rar-metadata/
    author:
      - ins: A. Zehavi
        name: Amir Zehavi
    date: 2026
  UMA2:
    title: "User-Managed Access (UMA) 2.0 Grant for OAuth 2.0 Authorization"
    target: https://docs.kantarainitiative.org/uma/wg/rec-oauth-uma-grant-2.0.html
    author:
      - ins: E. Maler
      - ins: M. Machulak
      - ins: J. Richer
    date: 2018
  OPENAPI:
    title: "OpenAPI Specification"
    target: https://spec.openapis.org/oas/latest.html
    date: false
  A2A:
    title: "Agent2Agent (A2A) Protocol"
    target: https://a2a-protocol.org/
    date: false
  I-D.oauth-identity-assertion-authz-grant:
    title: OAuth 2.0 Identity Assertion JWT Authorization Grant
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/
    author:
      - ins: A. Parecki
        name: Aaron Parecki
      - ins: K. McGuinness
        name: Karl McGuinness
      - ins: B. Campbell
        name: Brian Campbell
    date: 2026
  TOKEN-EXCHANGE-DISCOVERY:
    title: OAuth 2.0 Token Exchange Target Service Discovery
    target: https://datatracker.ietf.org/doc/draft-mcguinness-token-xchg-target-svc-disco/
    author:
      - ins: K. McGuinness
        name: Karl McGuinness
      - ins: A. Parecki
        name: Aaron Parecki
    date: 2026
  APISJSON:
    title: "APIs.json"
    target: https://apisjson.org/
    date: false
  MCP-AUTHORIZATION:
    title: "Model Context Protocol Authorization"
    target: https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
    date: 2025
  MCP-SERVER-CARD:
    title: "Model Context Protocol Server Card"
    target: https://modelcontextprotocol.io/community/server-card/charter
    date: false

--- abstract

This specification defines the Service Catalog Endpoint, a per-user, lightweight HTTP API that lets a client (such as an autonomous agent) discover, in a single request, the set of services a user is permitted to access, together with the metadata required to connect to each service and obtain the credentials needed to call it. After the user signs in to their identity provider, the client discovers the endpoint from the provider's metadata and retrieves the catalog scoped to that user. The catalog unifies human- and machine-readable service description (in the style of APIs.json), well-known service metadata, and one or more connection methods per service. A service may be a conventional HTTP API, a Model Context Protocol (MCP) server, or an Agent2Agent (A2A) agent. Each connection method separates how the client obtains a credential (for example, OAuth 2.0 Token Exchange or the authorization code grant) from how it presents that credential to the service (for example, a bearer token, an API key, or mutual TLS), reusing OpenAPI security scheme definitions for the latter; OAuth 2.0 is not assumed. Services are categorized so that an agent can find, for example, an email or calendar service by capability, and carry typed links (documentation, sign-up, terms, and others) using well-known link relation types. An agent can use the catalog to plan intent-scoped access before requesting any token.

--- middle

# Introduction

An autonomous agent acting on behalf of a user frequently needs to answer two questions before it can do useful work: *which services can this user reach?* and *how do I connect to a given service and obtain the credentials to call it?* Today, neither question has a standardized, user-scoped answer. Service inventories are maintained out of band (static configuration, proprietary catalogs, or human documentation), and connection details are discovered reactively, one service at a time.

Existing discovery mechanisms each solve part of the problem. OAuth 2.0 Protected Resource Metadata {{RFC9728}} lets a client discover the authorization server(s) for a *single* resource it already knows about, typically after an HTTP 401 challenge. OAuth 2.0 Authorization Server Metadata {{RFC8414}} describes a *single* authorization server. The Model Context Protocol authorization specification {{MCP-AUTHORIZATION}} composes these with Dynamic Client Registration {{RFC7591}} and Resource Indicators {{RFC8707}} into a per-service connection flow, and MCP Server Cards {{MCP-SERVER-CARD}} describe a single MCP server's capabilities. APIs.json {{APISJSON}} provides a "sitemap for APIs" for human and tooling consumption, but carries no per-user authorization context. None of these provides a single, user-scoped answer to *what is available to me and how do I connect*.

This specification defines the **Service Catalog Endpoint**: a single HTTP resource that returns, for the authenticated user, the set of services available to that user and, for each service, the descriptive metadata needed to understand it and one or more **connection methods** needed to obtain credentials and call it. The catalog is produced by a **Catalog Provider**, which MAY aggregate services that span multiple resources, multiple authorization servers, multiple authentication schemes, and multiple administrative domains.

The design is deliberately general:

* **Multiple service types.** A service may be a conventional HTTP API (`http`), an MCP server (`mcp`), or an A2A agent (`a2a`). MCP servers and A2A agents are first-class: such a service references its MCP Server Card {{MCP-SERVER-CARD}} or A2A Agent Card {{A2A}} so a client can learn the service's capabilities without connecting. Service types are extensible through an IANA registry.

* **Two-layer authentication, OAuth not assumed.** Each connection method separates *credential acquisition* (its `type` -- OAuth 2.0 token exchange, authorization code, client credentials, the Identity Assertion Authorization Grant {{I-D.oauth-identity-assertion-authz-grant}}, a pre-provisioned credential, or none) from *credential presentation* (how the credential is presented when calling the service, expressed as an OpenAPI {{OPENAPI}} security scheme such as a bearer token, API key, or mutual TLS). Acquisition types are extensible through an IANA registry; presentation reuses OpenAPI security schemes.

* **Capability-based discovery.** Each service may carry well-known **categories** (such as `email` or `calendar`), so an agent can find a service by what it does rather than by name.

* **Well-known links.** Each service may carry typed **links** (such as documentation, sign-up, terms of service, or the MCP Server Card) using link relation types {{RFC8288}}.

This specification generalizes target discovery for OAuth 2.0 Token Exchange {{RFC8693}}: token exchange is modeled as one connection method (`token_exchange`) among several. A client that only needs token exchange can use the catalog as a superset of, and an alternative to, {{TOKEN-EXCHANGE-DISCOVERY}}.

This specification provides the following benefits:

* **Proactive, aggregated discovery.** A single request returns every service the user can reach across resources, authentication schemes, and domains, rather than discovering one service at a time in reaction to authorization failures.

* **User-scoped results.** The catalog reflects real-time policy evaluation for the authenticated user, returning only services and connection methods the user (and the calling client) is permitted to use.

* **Actionable connection metadata.** Each connection method carries enough information for the client to execute the connection, or to deterministically fetch what is missing, without out-of-band configuration.

* **Intent-based planning.** An agent can read the catalog, and the capability descriptors it references, to plan for a user's goal and then request only the intent-scoped access it needs, including fine-grained access via Rich Authorization Requests {{RFC9396}} ({{intent}}).

* **Extensibility without new mechanisms.** Service types, categories, link relations, and connection methods are added through registries rather than by defining new protocols.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms:

Catalog Provider:
: The server that hosts the Service Catalog Endpoint and returns the catalog for the authenticated user. A Catalog Provider MAY describe services served by many resource servers, protected by many authorization servers, and using many authentication schemes. It is a role distinct from, and not necessarily co-located with, any service it describes.

Client:
: An application, such as an autonomous agent, that requests the catalog and subsequently connects to services on behalf of a user.

Service:
: Something a client can call on behalf of the user, such as an HTTP API or an MCP server. A service has a service type (see {{service-types}}).

Connection Method:
: A description of one way a client can obtain a credential and present it to call a service. A connection method separates two layers: an acquisition `type` (how the credential is obtained; see {{connection-type-registry}}) and a presentation (how the credential is presented to the service; see {{connection-object}}). OAuth 2.0 token issuance is one acquisition type; others use a pre-provisioned credential or none.

All members and string values defined by this document are case sensitive unless otherwise stated. All URIs are absolute URIs {{RFC8259}} unless otherwise stated.

# Service Catalog Endpoint

The Service Catalog Endpoint is an HTTP resource that returns the catalog of services available to the authenticated user. The client MUST use TLS as specified in {{Section 1.6 of RFC6749}}.

## Discovering the Catalog Endpoint {#endpoint-discovery}

The Service Catalog Endpoint is anchored to the user's identity provider: the OAuth 2.0 authorization server (or OpenID Connect Provider) that the user signs in to. After the user has authenticated, a client discovers and calls the endpoint as follows:

1. Determine the issuer. When the client has completed an OpenID Connect sign-in, the issuer is the `iss` value of the ID token. When the client starts from only a user identifier (for example, an `acct:` URI) and has not yet signed in, it MAY use WebFinger {{RFC7033}} to resolve the issuer, using the relation type `http://openid.net/specs/connect/1.0/issuer`, and then sign in. WebFinger is used only to resolve the issuer; it MUST NOT be used to convey the catalog endpoint or its contents (see {{privacy-considerations}}).

2. Read the issuer metadata. The client fetches the issuer's authorization server metadata {{RFC8414}} (an OpenID Connect deployment uses the OpenID Provider configuration, which reuses the same metadata registry) and reads the `service_catalog_endpoint` value ({{iana-as-metadata}}). The client MUST verify that the metadata `issuer` matches the expected issuer. If the metadata does not contain `service_catalog_endpoint`, the default location is the well-known URI {{RFC8615}} formed by appending `service-catalog` to `/.well-known/` of the issuer (that is, `{issuer}/.well-known/service-catalog`; see {{iana-well-known}}).

3. Retrieve the catalog. The client sends an authenticated request to the endpoint with the user's access token ({{catalog-request}}). The catalog is scoped to the user identified by the access token, not by the endpoint URL.

A client MAY instead be configured directly with the absolute URL of a Service Catalog Endpoint, in which case steps 1 and 2 are not used. The Catalog Provider that hosts the endpoint MAY be co-located with the authorization server or operated separately; when operated separately, the access token presented to the endpoint MUST be one the Catalog Provider accepts (for example, issued with the Catalog Provider as an audience).

## Catalog Request {#catalog-request}

The client retrieves the catalog by sending an HTTP `GET` request to the Service Catalog Endpoint.

The client MUST authenticate the request. The Catalog Provider determines the user on whose behalf the catalog is produced from the request credentials. A client SHOULD present an OAuth 2.0 access token using the `Authorization` request header field with the `Bearer` scheme {{RFC6750}}; other authentication methods supported by the Catalog Provider MAY be used. If the request is not authenticated and authentication is required, the Catalog Provider MUST return an HTTP 401 (Unauthorized) response.

A client MAY constrain the response with the filtering query parameters in {{filtering}}, and page through a large result with the pagination query parameters in {{pagination}}. A Catalog Provider MUST ignore any query parameter it does not support or understand. All query parameter names and values are case sensitive.

The following is an example request for the user's email services:

    GET /.well-known/service-catalog?category=email HTTP/1.1
    Host: catalog.example.com
    Authorization: Bearer SlAV32hkKG...ACCESSTOKEN...
    Accept: application/json

### Filtering {#filtering}

A client MAY include the following OPTIONAL query parameters to filter the services returned. When several different filter parameters are present, a service is returned only if it matches all of them (logical AND). When a single filter parameter is repeated, a service is returned if it matches any of the repeated values (logical OR), except for `tag` as noted below. Filtering is applied before pagination ({{pagination}}).

category:
: A well-known service category value (see {{service-categories}}), such as `email` or `calendar`. Only services in the given category are returned. MAY be repeated (OR).

type:
: A service type value (see {{service-types}}), such as `http` or `mcp`. Only services of the given type are returned. MAY be repeated (OR).

status:
: A connection status value (see {{connection-object}}): `connected`, `available`, `consent_required`, or `unavailable`. Only services that have at least one connection method with the given status are returned. MAY be repeated (OR). For example, `status=connected&status=available` returns services the client can use without user interaction.

tag:
: A free-form tag value (see the `tags` member in {{service-object}}). Only services that carry the given tag are returned. MAY be repeated; when repeated, a service is returned only if it carries all of the given tags (AND).

id:
: A service identifier (see the `id` member in {{service-object}}). Only the service with the given identifier is returned. MAY be repeated (OR). This supports refreshing one or more already-known services.

q:
: A free-text search string. The Catalog Provider MAY use this value to match services by name, description, or other provider-defined criteria.

A Catalog Provider SHOULD support the `category`, `type`, and `id` filters; support for the other filters is OPTIONAL. Because an unsupported filter is ignored rather than rejected, a client MUST NOT assume a filter was applied and SHOULD apply its own filtering to the result as needed.

### Pagination {#pagination}

A catalog may be large. A Catalog Provider MAY return services across multiple pages, and a client pages through them using the following OPTIONAL query parameters:

limit:
: A positive integer indicating the maximum number of services the client wishes to receive in a single response. The Catalog Provider MAY return fewer and MAY impose its own maximum page size; a `limit` greater than that maximum is treated as the maximum.

cursor:
: An opaque pagination cursor obtained from the `next_cursor` member (see {{catalog-object}}) of a previous response. The client MUST treat the cursor as opaque and MUST NOT modify it. A client that sends a `cursor` SHOULD repeat the same filter parameters it used to obtain that cursor; a Catalog Provider MAY reject a request that combines a `cursor` with inconsistent filters.

When more services match than are returned in a response, the Catalog Provider includes a `next_cursor` member in the catalog object ({{catalog-object}}); its absence indicates no further services are available. A Catalog Provider MAY also indicate the next page with a Link header field {{RFC8288}} using the relation type `next`. A Catalog Provider MAY paginate even when the client did not supply a `limit`.

The Catalog Provider MUST order services consistently across the pages of a single pagination sequence so that, absent concurrent changes, each matching service appears on exactly one page. A cursor MAY expire. A request with an expired or otherwise invalid `cursor` MUST be rejected with an HTTP 400 (Bad Request) and the error code `invalid_request` ({{error-response}}); the client recovers by restarting from the first page.

The following example requests the next page using a cursor from a previous response:

    GET /.well-known/service-catalog?cursor=b2Zmc2V0OjUw HTTP/1.1
    Host: catalog.example.com
    Authorization: Bearer SlAV32hkKG...ACCESSTOKEN...
    Accept: application/json

## Catalog Response {#catalog-response}

If the request is valid and authenticated, the Catalog Provider returns an HTTP 200 (OK) response whose body is a JSON {{RFC8259}} object, the **catalog object**. The `Content-Type` of the response MUST be `application/json`.

The Catalog Provider MUST evaluate the authenticated user's permissions, and the calling client's permissions, when constructing the catalog. The catalog MUST contain only services, and connection methods, that the user and client are permitted to use. The specific authorization policy evaluation is implementation specific. When constructing the response, the Catalog Provider MUST omit any member that would otherwise contain an empty string, an empty array, or a null value.

The Catalog Provider MAY include HTTP caching headers as specified in {{RFC9111}}. Because the catalog is user specific, any cache directives MUST mark the response as private (for example, `Cache-Control: private`).

### Catalog Object {#catalog-object}

The catalog object contains the following members:

services:
: REQUIRED. An array of **service objects** (see {{service-object}}), each describing a service available to the user. If the user has access to no services, this member is an empty array.

name:
: OPTIONAL. A human-readable name for the catalog.

description:
: OPTIONAL. A human-readable description of the catalog.

updated:
: OPTIONAL. The timestamp at which the catalog content was last updated, as an {{RFC3339}} date-time string.

next_cursor:
: OPTIONAL. An opaque string. If present, more services are available than were returned; the client obtains the next page by repeating the request with the `cursor` query parameter set to this value (see {{pagination}}). If absent, no further services are available.

Extensions MAY define additional members of the catalog object. Clients MUST ignore members they do not understand.

### Service Object {#service-object}

A service object describes a single service and how to connect to it. It contains the following members:

id:
: REQUIRED. A stable, provider-assigned identifier for the service, unique within the catalog. Clients MAY use this value to correlate a service across catalog requests.

name:
: REQUIRED. A human-readable name for the service, suitable for display to an end user (for example, in a service picker).

connections:
: REQUIRED. A non-empty array of **connection objects** (see {{connection-object}}), each describing one way the client can obtain credentials to call the service. The Catalog Provider SHOULD order the array from most to least preferred.

type:
: OPTIONAL. The service type, as a value from the registry in {{service-types}}. If omitted, the type is `http`.

description:
: OPTIONAL. A human-readable description of the service.

categories:
: OPTIONAL. An array of well-known service category values (see {{service-categories}}) describing what the service does, enabling capability-based discovery.

tags:
: OPTIONAL. An array of free-form tag strings categorizing the service.

links:
: OPTIONAL. An array of **link objects** (see {{link-object}}) pointing to related resources such as documentation, sign-up, or the MCP Server Card, using link relation types {{RFC8288}}.

base_uri:
: OPTIONAL. The primary endpoint URL at which the service is hosted: for an `http` service, the base URL of its API; for an `mcp` service, the MCP server endpoint. This is the network location of the service and is independent of how the client authenticates to it.

mcp:
: OPTIONAL. Present only when `type` is `mcp`; see {{type-mcp}}.

a2a:
: OPTIONAL. Present only when `type` is `a2a`; see {{type-a2a}}.

The service object intentionally carries no authentication-scheme-specific members. Values such as the authorization server, scopes, and resource indicator are properties of a particular authentication scheme, so they appear on the relevant connection object ({{connection-object}}) rather than on the service. A service that supports more than one authentication scheme has more than one connection object, each carrying the values for its scheme.

Extensions and service types MAY define additional members of the service object. Clients MUST ignore members they do not understand.

### Link Object {#link-object}

A link object points to a resource related to a service. It is modeled on the Web Linking framework {{RFC8288}} and contains the following members:

rel:
: REQUIRED. A link relation type {{RFC8288}}. The value is either a registered relation type or an extension relation type (an absolute URI). Relation types commonly used in a catalog include `service` (the service's primary endpoint or home), `service-desc` (a machine-readable API description such as an OpenAPI document), `service-doc` (human-readable documentation), `terms-of-service`, `privacy-policy`, `help`, `status` (a service status page), `icon`, `describedby` (for example, a Protected Resource Metadata document {{RFC9728}}), `sign-up` (where a user can sign up for the service), `mcp-server-card` (the service's MCP Server Card {{MCP-SERVER-CARD}}), and `agent-card` (the service's A2A Agent Card {{A2A}}). The relation types `sign-up`, `mcp-server-card`, and `agent-card` are registered by this document (see {{iana-link-relations}}).

href:
: REQUIRED. The target URL of the link.

type:
: OPTIONAL. A hint of the media type of the link target.

title:
: OPTIONAL. A human-readable title for the link.

### Connection Object {#connection-object}

A connection object describes one connection method for a service, separating two layers: how the client *acquires* a credential (its `type`) and how it *presents* that credential when calling the service (its `present` member, or a security scheme referenced from the service's descriptor). A connection object MUST carry enough information for the client either to execute the connection directly or to deterministically fetch the remaining information it needs. Heavy or detailed metadata (for example, the full authorization server metadata document) is referenced by URL rather than inlined.

A connection object contains the following members:

type:
: REQUIRED. The credential acquisition method: a string drawn from the "Service Catalog Connection Type" registry ({{connection-type-registry}}). This document defines `token_exchange`, `authorization_code`, `client_credentials`, `id_jag`, `pre_authorized`, and `none`.

present:
: OPTIONAL. How the client presents its credential when calling the service (the service-authentication layer, as distinct from the acquisition `type`). Its value is an OpenAPI Security Scheme Object {{OPENAPI}} -- the same model used by OpenAPI `securitySchemes` and A2A Agent Cards {{A2A}} -- for example `{"type": "http", "scheme": "bearer"}` for a bearer token, `{"type": "apiKey", "in": "header", "name": "X-API-Key"}` for an API key, or `{"type": "mutualTLS"}` for a client certificate. When `present` is omitted: for acquisition types that yield an access token (`token_exchange`, `authorization_code`, `client_credentials`, `id_jag`), the credential is presented as an HTTP bearer token {{RFC6750}}; otherwise the client determines presentation from the service's referenced security schemes. The catalog SHOULD NOT inline a `present` value that merely restates the security schemes already published in the service's descriptor (an OpenAPI {{OPENAPI}} document via `service-desc`, an A2A Agent Card, or an MCP Server Card); a client obtains those by reference ({{intent}}).

status:
: OPTIONAL. The per-user state of this connection method, as one of the following string values. If omitted, the client SHOULD treat the status as `available`.

    * `connected`: A valid credential already exists for the user; the client may be able to call the service without obtaining a new credential, or can refresh without user interaction.
    * `available`: The client can obtain a credential using this method without further user interaction (for example, a token exchange or client credentials grant).
    * `consent_required`: Obtaining a credential requires user interaction (for example, an authorization code grant with user consent).
    * `unavailable`: The method is described for completeness but cannot currently be used by this user.

The following members carry the OAuth 2.0 values for connection methods that obtain a token from an OAuth 2.0 authorization server (`token_exchange`, `authorization_code`, `client_credentials`, and `id_jag`). Because these values are specific to the OAuth scheme, they appear on the connection object, not on the service object; a connection object is self-contained and the client does not combine it with values from elsewhere in the catalog.

authorization_server:
: REQUIRED for the OAuth-based connection types. The issuer identifier {{RFC8414}} of the authorization server to use for this connection. The client obtains the authorization server's endpoints and capabilities from its metadata {{RFC8414}}.

token_endpoint:
: OPTIONAL. The authorization server's token endpoint URL. When present, the client MAY use it directly; when absent, the client obtains it from the authorization server metadata {{RFC8414}}.

resource:
: OPTIONAL. The resource indicator {{RFC8707}} the client includes when requesting a token for this connection, as a canonical URI per {{Section 2 of RFC8707}}. This identifies the protected resource and corresponds to the resource described by a Protected Resource Metadata document {{RFC9728}}, which MAY be referenced by a `describedby` link ({{link-object}}).

scopes:
: OPTIONAL. An array of OAuth 2.0 scope values {{Section 3.3 of RFC6749}} to request for this connection.

authorization_details_types:
: OPTIONAL. An array of OAuth 2.0 Rich Authorization Requests {{RFC9396}} `authorization_details` type identifiers (strings) that this connection accepts for this user, enabling fine-grained, intent-scoped access requests independently of `scopes`. This is the per-service, per-user counterpart to the server-wide `authorization_details_types_supported` metadata of {{RFC9396}}: it tells the client which types are usable here, while the schema, documentation, and examples for each type are obtained from the authorization server's Authorization Details Types Metadata {{RAR-METADATA}} (discovered via this connection's `authorization_server`) rather than restated in the catalog.

client_id:
: OPTIONAL. A static OAuth 2.0 client identifier the client uses with the `authorization_server`. This supports authorization servers that require a pre-registered, per-service, or per-tenant client registration.

client_registration:
: OPTIONAL. How the client obtains a client identifier when `client_id` is absent: `dynamic` (register using OAuth 2.0 Dynamic Client Registration {{RFC7591}}, discovering the registration endpoint from the authorization server metadata) or `none` (no client identifier is required). When both `client_id` and `client_registration` are absent, the client uses a client identity it determines is appropriate by other means.

Connection types define additional type-specific members, as described in {{connection-types}}. Extensions MAY define additional members of the connection object. Clients MUST ignore members they do not understand.

## Service Types {#service-types}

The `type` member of a service object identifies the kind of service, using a value from the "Service Catalog Service Type" registry ({{iana-service-type}}). This document defines three values.

### http {#type-http}

A conventional HTTP API. The `base_uri` member is the base URL of the API. A machine-readable description (for example, an OpenAPI document) MAY be referenced with a `service-desc` link ({{link-object}}). This is the default type when `type` is omitted.

### mcp {#type-mcp}

A Model Context Protocol server {{MCP-AUTHORIZATION}}. The `base_uri` member is the MCP server endpoint. A service of type `mcp` SHOULD reference the server's MCP Server Card {{MCP-SERVER-CARD}}, either with an `mcp-server-card` link ({{link-object}}) or with the `server_card_uri` member below, so that a client can learn the server's capabilities without connecting.

A service of type `mcp` MAY include an `mcp` member, a JSON object with the following members:

transport:
: OPTIONAL. The MCP transport the endpoint uses (for example, `streamable-http`).

server_card_uri:
: OPTIONAL. The URL of the server's MCP Server Card {{MCP-SERVER-CARD}}.

Because MCP servers use OAuth 2.0 for authorization {{MCP-AUTHORIZATION}}, an `mcp` service typically offers OAuth-based connection methods (for example, `authorization_code` or `token_exchange`); service type and connection type are independent.

### a2a {#type-a2a}

An Agent2Agent (A2A) agent {{A2A}}. The `base_uri` member is the A2A agent endpoint. A service of type `a2a` SHOULD reference the agent's A2A Agent Card, either with an `agent-card` link ({{link-object}}) or with the `agent_card_uri` member below, so that a client can learn the agent's skills and capabilities without connecting.

A service of type `a2a` MAY include an `a2a` member, a JSON object with the following members:

transport:
: OPTIONAL. The A2A transport the endpoint uses (for example, `JSONRPC`).

agent_card_uri:
: OPTIONAL. The URL of the agent's A2A Agent Card (typically at `/.well-known/agent-card.json`).

As with `mcp`, service type and connection type are independent; an `a2a` agent describes how callers authenticate to it in its Agent Card, which the catalog references rather than restates.

## Service Categories {#service-categories}

The `categories` member of a service object, and the `category` query parameter ({{catalog-request}}), use well-known category values from the "Service Catalog Service Category" registry ({{iana-category}}) to describe what a service does. This document seeds the registry with the values `email`, `calendar`, `contacts`, `files`, `chat`, `tasks`, `crm`, and `ticketing`. The registry is extensible; an agent can rely on these values to find a service by capability (for example, "an email service") rather than by name.

## Connection Types {#connection-types}

This section defines the initial credential acquisition types -- the values of a connection's `type`. How the resulting credential is presented to the service is a separate layer, given by the connection's `present` member ({{connection-object}}). All reuse the common connection object members from {{connection-object}}; the members below are type specific.

### token_exchange {#type-token-exchange}

The client obtains a token by performing an OAuth 2.0 Token Exchange {{RFC8693}} at the `authorization_server`, presenting a token it already holds for the user as the `subject_token`. This is the catalog equivalent of {{TOKEN-EXCHANGE-DISCOVERY}}.

Type-specific members:

audience:
: OPTIONAL. The `audience` value {{Section 2.1 of RFC8693}} to include in the token exchange request. The `audience` is the logical name of the target service and need not be a URI; it MAY be an opaque, authorization-server-local identifier. For a multi-tenant service, the Catalog Provider returns a distinct connection object per tenant, each with a distinct `audience` value that selects the tenant.

tenant:
: OPTIONAL. A machine-readable identifier for the tenant of a multi-tenant service. This member is descriptive (it lets the client correlate the connection with the tenant context of the resulting token) and is not used as a request parameter; the `audience` value selects the tenant.

supported_token_types:
: OPTIONAL. An array of token type URIs {{RFC8693}} that may be requested. If omitted, the client may request any token type the authorization server supports.

### authorization_code {#type-authorization-code}

The client obtains a token using the OAuth 2.0 authorization code grant {{Section 4.1 of RFC6749}} with PKCE. This type typically has `status` of `consent_required`. The client uses the `authorization_server` metadata {{RFC8414}} to locate endpoints, includes the `resource` parameter {{RFC8707}}, and uses `client_id` or `client_registration` to determine its client identity. This type defines no additional members.

### client_credentials {#type-client-credentials}

The client obtains a token using the OAuth 2.0 client credentials grant {{Section 4.4 of RFC6749}}, authenticating as itself. This type typically has `status` of `available`. This type defines no additional members.

### id_jag {#type-id-jag}

The client obtains an access token by redeeming an Identity Assertion Authorization Grant (ID-JAG) {{I-D.oauth-identity-assertion-authz-grant}} at the service's `authorization_server`. The client first obtains an ID-JAG for the user from the user's identity provider and then redeems it.

Type-specific members:

audience:
: OPTIONAL. The audience value identifying the service for which the ID-JAG is requested.

tenant:
: OPTIONAL. A machine-readable identifier for the tenant of a multi-tenant service, with the same descriptive semantics as in {{type-token-exchange}}.

### none {#type-none}

The service requires no client authentication (it is public). This type defines no additional members and typically has `status` of `available`.

### pre_authorized {#type-pre-authorized}

The credential is one the client already possesses out of band (for example, a long-lived token, an API key, or a client certificate provisioned to the client). The catalog indicates that such a method exists but MUST NOT include the credential. How the credential is presented is given by the connection's `present` member ({{connection-object}}). This type typically has `status` of `connected`.

A Catalog Provider MUST NOT include secret values (access tokens, refresh tokens, client secrets, API keys, or private keys) anywhere in the catalog.

## Catalog Response Example {#response-example}

The following is a non-normative example response showing four services: an HTTP service in the `email` category offering both token exchange and user consent, an MCP server referencing its Server Card, a service the user is already connected to via a pre-provisioned API key (note the `present` member), and an A2A agent referencing its Agent Card.

    HTTP/1.1 200 OK
    Content-Type: application/json
    Cache-Control: private, max-age=60

    {
      "name": "Example Workspace",
      "updated": "2026-06-16T18:00:00Z",
      "services": [
        {
          "id": "mail",
          "name": "Example Mail",
          "type": "http",
          "categories": ["email"],
          "base_uri": "https://api.example.com/mail",
          "links": [
            {"rel": "service-doc",
             "href": "https://dev.example.com/mail"},
            {"rel": "service-desc",
             "href": "https://api.example.com/mail/openapi.json"},
            {"rel": "sign-up", "href": "https://example.com/signup"}
          ],
          "connections": [
            {
              "type": "token_exchange",
              "status": "available",
              "authorization_server": "https://as.example.com",
              "resource": "https://api.example.com/mail",
              "scopes": ["mail.read", "mail.send"]
            },
            {
              "type": "authorization_code",
              "status": "consent_required",
              "authorization_server": "https://as.example.com",
              "resource": "https://api.example.com/mail",
              "scopes": ["mail.read", "mail.send"],
              "authorization_details_types": ["mail.message"],
              "client_registration": "dynamic"
            }
          ]
        },
        {
          "id": "tickets",
          "name": "Support Tickets (MCP)",
          "type": "mcp",
          "categories": ["ticketing"],
          "base_uri": "https://mcp.example.com/mcp",
          "mcp": {"transport": "streamable-http"},
          "links": [
            {"rel": "mcp-server-card",
             "href": "https://mcp.example.com/.well-known/mcp.json"}
          ],
          "connections": [
            {
              "type": "authorization_code",
              "status": "consent_required",
              "authorization_server": "https://as.example.com",
              "resource": "https://mcp.example.com/mcp",
              "client_registration": "dynamic"
            }
          ]
        },
        {
          "id": "calendar",
          "name": "Calendar API",
          "categories": ["calendar"],
          "base_uri": "https://api.calendar.example",
          "connections": [
            {
              "type": "pre_authorized",
              "status": "connected",
              "present": {
                "type": "apiKey", "in": "header", "name": "X-API-Key"
              }
            }
          ]
        },
        {
          "id": "scheduler",
          "name": "Scheduler Agent",
          "type": "a2a",
          "categories": ["calendar"],
          "base_uri": "https://agent.example/a2a",
          "a2a": {"transport": "JSONRPC"},
          "links": [
            {"rel": "agent-card",
             "href":
               "https://agent.example/.well-known/agent-card.json"}
          ],
          "connections": [
            {
              "type": "authorization_code",
              "status": "consent_required",
              "authorization_server": "https://as.example.com",
              "resource": "https://agent.example/a2a",
              "client_registration": "dynamic"
            }
          ]
        }
      ]
    }

## Error Response {#error-response}

If the request fails, the Catalog Provider returns an HTTP error response whose body, when present, is a JSON object using the error format of {{Section 5.2 of RFC6749}} (`error` and OPTIONAL `error_description`). The HTTP status code is set as follows:

* 401 (Unauthorized): authentication is required and was missing or invalid.
* 403 (Forbidden): the authenticated client is not authorized to use the catalog endpoint.
* 400 (Bad Request): the request is malformed, or includes an invalid or expired `cursor` (see {{pagination}}).
* 503 (Service Unavailable): the Catalog Provider is temporarily unable to handle the request.

The following is an example error response:

    HTTP/1.1 401 Unauthorized
    Content-Type: application/json
    WWW-Authenticate: Bearer

    {
      "error": "invalid_token",
      "error_description": "The access token is missing or expired"
    }

# Connecting to a Service

After retrieving the catalog, a client connects to a service by selecting a service object and one of its connection objects, and then executing the corresponding connection type. The general procedure is:

1. Select a service (for example, by `category`, `tags`, or by presenting `name` and `description` to the user). For an `mcp` service, the client MAY first fetch the MCP Server Card ({{type-mcp}}) to evaluate the server's capabilities.
2. Select a connection object. A client SHOULD prefer a connection whose `status` is `connected` or `available` over one whose `status` is `consent_required`, when more than one is suitable.
3. For OAuth-based connection types, determine the client identity: use `client_id` if present; otherwise, if `client_registration` is `dynamic`, register with the authorization server using {{RFC7591}}; otherwise proceed without a client identifier if `client_registration` is `none`.
4. Obtain the credential according to the connection `type` (see {{connection-types}}), using the values on the selected connection object, including the `resource` {{RFC8707}} and `scopes` for OAuth-based types.
5. Call the service, presenting the credential as described by the connection's `present` member or, when it is absent, by the service's referenced security schemes (an OpenAPI {{OPENAPI}} document, an A2A Agent Card, or an MCP Server Card) -- for example, as a bearer token {{RFC6750}}, an API key, or a client certificate.

This specification does not define any new credential issuance mechanism. Each connection type parameterizes an existing mechanism.

# Intent-Based Use and Resource Discovery {#intent}

An agent typically uses the catalog to plan: it discovers the services a user can reach, decides which it needs for the user's goal, and requests only the access that goal requires. The catalog supports planning before commitment in two ways:

* Plan from descriptors without connecting. A service object's `links` reference the service's capability descriptor -- for example, an OpenAPI {{OPENAPI}} document via a `service-desc` link, or an MCP Server Card {{MCP-SERVER-CARD}} via an `mcp-server-card` link. A client MAY read these to understand a service's operations and resources, and plan, without obtaining any token.

* Connect with minimal access to enumerate, then request what is needed. When an agent must inspect the user's actual resources before planning (for example, to list the user's mailboxes or calendars), it SHOULD select a connection and request the least-privilege access sufficient to enumerate -- narrow `scopes`, or a minimal `authorization_details` request limited to the `authorization_details_types` the connection advertises ({{connection-object}}) -- and then make a further, intent-scoped request for the access it determines it needs.

Incremental, intent-scoped requests keep each token least-privilege. To support them:

* When a connection advertises `authorization_details_types` and the authorization server offers a pushed authorization request endpoint {{RFC9126}}, the client SHOULD push the (potentially large) `authorization_details` request rather than place it in a front-channel URL.

* A client MUST be prepared for runtime escalation signals it cannot fully predict from the catalog: a resource server MAY indicate that stronger or fresher user authentication is required {{RFC9470}}, or that the presented authorization details are insufficient {{RAR-METADATA}}. In each case the client repeats the request with the indicated requirements.

The catalog is an authorization-filtered overlay: it lists what a user can reach and how to connect, but it does not republish a service's full capability surface. A client obtains the authoritative, and possibly authorization-dependent, capability list from the service itself (for example, an MCP server's runtime tool list) or from a referenced descriptor, not from the catalog.

# Relationship to Other Work {#related-work}

The Service Catalog occupies a layer that existing mechanisms do not: a per-user, authorization-filtered overlay that enumerates a user's reachable services and how to connect to each. It composes with, rather than replaces, the following.

OAuth 2.0 Token Exchange Target Service Discovery {{TOKEN-EXCHANGE-DISCOVERY}} discovers the audiences, resources, and scopes a subject token may be exchanged for. It is specialized to token exchange; the `token_exchange` connection type ({{type-token-exchange}}) carries the same information as one connection method among several. The two are independent and this document does not depend on it.

Rich Authorization Requests {{RFC9396}} and its metadata extension {{RAR-METADATA}} express and describe fine-grained authorization details, but their metadata is server-wide. The catalog advertises, per service and per user, which `authorization_details` types are usable ({{connection-object}}) and defers each type's schema and documentation to {{RAR-METADATA}}.

Grant Negotiation and Authorization Protocol (GNAP) {{RFC9635}} negotiates the access a client declares it wants at a single grant endpoint; its intent is client-declared, not a server-advertised enumeration of what a user can reach. A catalog entry's `id` and connection values name what a client can subsequently request, whether by token exchange, an OAuth grant, or a GNAP access request.

User-Managed Access (UMA 2.0) {{UMA2}} lets a resource owner share resources with other parties; its resource registration is resource-server-to-authorization-server and is not client-facing, and its resource list is a synchronization aid rather than a browsable, per-user catalog. The catalog fills that client-facing gap.

Agent and tool descriptors -- A2A Agent Cards {{A2A}}, MCP Server Cards {{MCP-SERVER-CARD}}, and machine-readable API descriptions such as OpenAPI {{OPENAPI}} -- describe a single service instance, including its capabilities and how it authenticates callers. The catalog references these (via `links`) rather than duplicating them, and adds the per-user authorization context they lack.

APIs.json {{APISJSON}} provides a public "sitemap for APIs" with no per-user authorization context. A service object reuses its discoverability model: `name`, `description`, `base_uri` (its `baseURL`), `tags`, and `links` (its typed `properties`). A Catalog Provider MAY additionally serve a static APIs.json document for tooling that consumes that format; such a document is out of scope for this specification.

# Security Considerations

## Authentication and Transport

The Service Catalog Endpoint MUST be served over TLS {{Section 1.6 of RFC6749}}. The Catalog Provider MUST authenticate the request and produce the catalog only for the authenticated user. Credentials MUST NOT be placed in the request URI.

## Information Disclosure

The catalog reveals the complete set of services a user can reach, together with the authentication schemes and connection methods for each. This is strictly more information than a per-service discovery and is valuable to an attacker for reconnaissance. To mitigate this risk:

* The Catalog Provider MUST require authentication and return only services and connection methods the authenticated user and client are authorized to use.
* The Catalog Provider SHOULD apply rate limiting and SHOULD log access for security monitoring.
* Responses are user specific and MUST NOT be cached by shared caches; cache directives, when present, MUST mark the response as private.

## No Secrets in the Catalog

A Catalog Provider MUST NOT include secret values (access tokens, refresh tokens, client secrets, API keys, or private keys) in the catalog. Connection objects describe how to obtain or present a credential, not the credential itself. Consistent with {{MCP-SERVER-CARD}}, a referenced MCP Server Card likewise MUST NOT contain secrets; a client treats the card as untrusted input and validates it.

## Token Audience Binding and Passthrough

For OAuth-based connection methods, a client MUST request tokens scoped to the specific service it intends to call, including the `resource` indicator {{RFC8707}}, so that issued tokens are bound to their intended service. A client MUST NOT present a token issued for one service to a different service. A service acting as a client to a further upstream service MUST obtain a separate token rather than passing through the token it received. These requirements mirror the token audience and passthrough guidance in {{MCP-AUTHORIZATION}}.

## Trust in the Catalog Provider

A client trusts the Catalog Provider to enumerate services, endpoints, and connection methods accurately. A malicious or compromised Catalog Provider could induce a client to connect to an attacker-controlled service, authorization server, or MCP server, or to present credentials to the wrong party. A client SHOULD obtain the Catalog Provider's base URL from a trusted source ({{endpoint-discovery}}), MUST validate that each service's `base_uri` and each connection's `authorization_server` and `resource` are acceptable under its policy before connecting, and SHOULD apply the same scrutiny to resources referenced by `links` (such as a `service-desc` or `mcp-server-card`).

## Confused Deputy and Client Registration

When `client_registration` is `dynamic` ({{RFC7591}}), a client registers with authorization servers it may not have known in advance. A client MUST apply appropriate consent and validation before registering with, or obtaining tokens from, an authorization server discovered through the catalog, consistent with the confused-deputy guidance in {{MCP-AUTHORIZATION}}.

# Privacy Considerations {#privacy-considerations}

The catalog is inherently personal: it describes the services a specific user can access. This may reveal organizational affiliations, tenant memberships, and the user's relationships to third-party services. To protect privacy:

* The catalog, and any per-user catalog endpoint URL, MUST be obtained through an authenticated channel ({{endpoint-discovery}}). In particular, WebFinger {{RFC7033}} MUST NOT be used to convey the catalog endpoint or its contents: WebFinger responses are unauthenticated and, by design, broadly accessible (including cross-origin), so publishing the set of services a user can reach there would disclose personal data to anyone. WebFinger is used only to resolve a user identifier to an issuer.

* The Catalog Provider SHOULD return only information the authenticated user and client are authorized to know, applying the principle of least privilege.
* The Catalog Provider SHOULD NOT include identifiers of other users or of tenants the user is not associated with.
* The Catalog Provider SHOULD log access in accordance with applicable privacy regulations and SHOULD minimize retention of catalog requests.
* A deployment MAY provide mechanisms for users to review and limit the services exposed through the catalog.

# IANA Considerations

## OAuth Authorization Server Metadata {#iana-as-metadata}

IANA is requested to register the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata Name: `service_catalog_endpoint`

Metadata Description: URL of the per-user Service Catalog Endpoint associated with this authorization server.

Change Controller: IETF

Specification Document(s): [[ this document ]]

## Well-Known URI {#iana-well-known}

IANA is requested to register the following well-known URI in the "Well-Known URIs" registry established by {{RFC8615}}.

URI Suffix: `service-catalog`

Change Controller: IETF

Specification Document(s): [[ this document ]]

Status: permanent

## Service Catalog Service Type Registry {#iana-service-type}

IANA is requested to establish the "Service Catalog Service Type" registry for values of the `type` member of a service object ({{service-object}}). The registration policy is Specification Required. Each registration contains the type value, a brief description, and a reference. Values are case-sensitive strings of printable ASCII characters without whitespace. Initial contents:

| Service Type | Description | Reference |
| ------------ | ----------- | --------- |
| `http` | HTTP API | {{type-http}} |
| `mcp` | Model Context Protocol server | {{type-mcp}} |
| `a2a` | Agent2Agent agent | {{type-a2a}} |

## Service Catalog Service Category Registry {#iana-category}

IANA is requested to establish the "Service Catalog Service Category" registry for values of the `categories` member of a service object and the `category` query parameter ({{service-categories}}). The registration policy is First Come First Served. Each registration contains the category value, a brief description, and a contact or reference. Values are case-sensitive strings of printable ASCII characters without whitespace. Initial contents:

| Category | Description | Reference |
| -------- | ----------- | --------- |
| `email` | Electronic mail | {{service-categories}} |
| `calendar` | Calendaring and scheduling | {{service-categories}} |
| `contacts` | Contact and address book | {{service-categories}} |
| `files` | File storage | {{service-categories}} |
| `chat` | Messaging and chat | {{service-categories}} |
| `tasks` | Tasks and to-do | {{service-categories}} |
| `crm` | Customer relationship management | {{service-categories}} |
| `ticketing` | Support ticketing and issue tracking | {{service-categories}} |

## Service Catalog Connection Type Registry {#connection-type-registry}

IANA is requested to establish the "Service Catalog Connection Type" registry for values of the `type` member of a connection object ({{connection-object}}). The registration policy is Specification Required. Each registration contains the connection type value, a brief description, the type-specific members it defines (if any), and a reference. Values are case-sensitive strings of printable ASCII characters without whitespace. Initial contents:

| Connection Type | Description | Reference |
| --------------- | ----------- | --------- |
| `token_exchange` | OAuth 2.0 Token Exchange | {{type-token-exchange}} |
| `authorization_code` | Authorization code grant with PKCE | {{type-authorization-code}} |
| `client_credentials` | Client credentials grant | {{type-client-credentials}} |
| `id_jag` | Identity Assertion Authorization Grant | {{type-id-jag}} |
| `pre_authorized` | Pre-provisioned out-of-band credential | {{type-pre-authorized}} |
| `none` | No credential required (public) | {{type-none}} |

## Link Relation Types {#iana-link-relations}

IANA is requested to register the following relation types in the "Link Relation Types" registry established by {{RFC8288}}.

Relation Name: `sign-up`

Description: Refers to a resource where a user can sign up for the linked service.

Reference: [[ this document ]]

Relation Name: `mcp-server-card`

Description: Refers to the Model Context Protocol Server Card for the linked service.

Reference: [[ this document ]]

Relation Name: `agent-card`

Description: Refers to the Agent2Agent (A2A) Agent Card for the linked service.

Reference: [[ this document ]]

--- back

# Document History
{:numbered="false"}

-00

* Initial revision.

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
