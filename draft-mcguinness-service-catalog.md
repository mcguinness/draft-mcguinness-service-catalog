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
  RFC6838:
  RFC8259:
  RFC8288:
  RFC8414:
  RFC8615:
  RFC8707:
  RFC9110:
  RFC9111:
  RFC9396:
  RFC9457:
  RFC9728:

informative:
  RFC7033:
  RFC7591:
  RFC8693:
  RFC9126:
  RFC9207:
  RFC9449:
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
  ARAZZO:
    title: "Arazzo Specification"
    target: https://spec.openapis.org/arazzo/latest.html
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

This specification defines the Service Catalog Endpoint, a per-user HTTP API that lets a client (such as an autonomous agent) discover, in a single request, the services a user can reach and how to connect to each. After the user signs in to their identity provider, the client discovers the endpoint from the provider's metadata and retrieves the catalog scoped to that user. A service may be a conventional HTTP API, a Model Context Protocol (MCP) server, or an Agent2Agent (A2A) agent, and each connection method separates how a credential is obtained (for example, OAuth 2.0 Token Exchange or the authorization code grant) from how it is presented (for example, a bearer token, an API key, or mutual TLS); OAuth 2.0 is not assumed. The catalog reuses APIs.json-style description, well-known link relation types, and OpenAPI security schemes, and lets an agent plan intent-scoped access before requesting any token.

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

## What the Catalog Is Not

The catalog is deliberately narrow. It is **not**:

* authoritative API or capability metadata -- that lives in the service's OpenAPI document, MCP Server Card, or A2A Agent Card, which the catalog references rather than duplicates ({{related-work}});
* a token issuance or grant protocol -- it describes how to use existing mechanisms and issues no credential itself ({{connecting}});
* proof of current authorization -- a listed service, or a connection's `connected` status, does not guarantee a subsequent call will succeed ({{availability}});
* a substitute for OAuth 2.0 Protected Resource Metadata or Authorization Server Metadata -- those remain the authoritative source for endpoints and policy, and a client re-anchors to them before acting ({{connection-object}}).

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
: One way a client can obtain a credential and present it to call a service. A connection method separates two layers: an acquisition `type` (how the credential is obtained; see {{connection-type-registry}}) and a presentation (how the credential is presented to the service; see {{connection-object}}). OAuth 2.0 token issuance is one acquisition type; others use a pre-provisioned credential or none.

All members and string values defined by this document are case sensitive unless otherwise stated. All URIs are absolute URIs {{RFC8259}} unless otherwise stated.

# Overview {#processing-model}

This section summarizes how a client uses the catalog end to end; the sections that follow specify each step.

1. Discover the catalog endpoint ({{endpoint-discovery}}).
2. Authenticate to the catalog and retrieve it ({{catalog-request}}).
3. Filter and select a service ({{filtering}}, {{service-object}}).
4. Verify the selected service's resource and authorization server metadata, re-anchoring trust to the resource ({{connection-object}}).
5. Select a connection method ({{connection-object}}).
6. Obtain the credential for the selected connection ({{connecting}}).
7. Call the service, presenting the credential ({{connecting}}).

# The Service Catalog Endpoint

The Service Catalog Endpoint is an HTTP resource that returns the catalog of services available to the authenticated user. The client MUST use TLS as specified in {{Section 1.6 of RFC6749}}.

## Discovering the Catalog Endpoint {#endpoint-discovery}

The Service Catalog Endpoint is anchored to the user's identity provider: the OAuth 2.0 authorization server (or OpenID Connect Provider) that the user signs in to. After the user has authenticated, a client discovers and calls the endpoint as follows:

1. Determine the issuer. When the client has completed an OpenID Connect sign-in, the issuer is the `iss` value of the ID token. When the client starts from only a user identifier (for example, an `acct:` URI) and has not yet signed in, it MAY use WebFinger {{RFC7033}} to resolve the issuer, using the relation type `http://openid.net/specs/connect/1.0/issuer`, and then sign in. WebFinger is used only to resolve the issuer; it MUST NOT be used to convey the catalog endpoint or its contents (see {{privacy-considerations}}).

2. Read the issuer metadata. The client fetches the issuer's authorization server metadata {{RFC8414}} (an OpenID Connect deployment uses the OpenID Provider configuration, which reuses the same metadata registry) and reads the `service_catalog_endpoint` value ({{iana-as-metadata}}), having verified that the metadata `issuer` matches the expected issuer.

    * The metadata value is authoritative; the client SHOULD prefer it when present.
    * If the metadata omits `service_catalog_endpoint`, a provider co-located with the issuer MAY serve the endpoint at `{issuer}/.well-known/service-catalog` {{RFC8615}} (see {{iana-well-known}}); this fallback is an optional deployment convenience, not a requirement.
    * A client MUST NOT treat a catalog at an origin different from the issuer as authoritative for the issuer's users on the basis of URL structure alone; cross-origin authority requires the metadata value or explicit configuration.

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

A Catalog Provider SHOULD support the `category`, `type`, and `id` filters; support for the other filters is OPTIONAL. A Catalog Provider that paginates ({{pagination}}) MUST support the `category`, `type`, and `id` filters, so that a client can bound a large result at the server rather than retrieving and locally filtering every page. Because an unsupported optional filter is ignored rather than rejected, a client MUST NOT assume such a filter was applied and SHOULD apply its own filtering to the result as needed.

### Pagination {#pagination}

A catalog may be large. A Catalog Provider MAY return services across multiple pages, and a client pages through them using the following OPTIONAL query parameters:

limit:
: A positive integer indicating the maximum number of services the client wishes to receive in a single response. The Catalog Provider MAY return fewer and MAY impose its own maximum page size; a `limit` greater than that maximum is treated as the maximum.

cursor:
: An opaque pagination cursor obtained from the `next_cursor` member (see {{catalog-object}}) of a previous response. The client MUST treat the cursor as opaque and MUST NOT modify it. A client that sends a `cursor` SHOULD repeat the same filter parameters it used to obtain that cursor; a Catalog Provider MAY reject a request that combines a `cursor` with inconsistent filters.

When more services match than are returned in a response, the Catalog Provider includes a `next_cursor` member in the catalog object ({{catalog-object}}); its absence indicates no further services are available. A Catalog Provider MAY also indicate the next page with a Link header field {{RFC8288}} using the relation type `next`. A Catalog Provider MAY paginate even when the client did not supply a `limit`.

The Catalog Provider MUST order services consistently across the pages of a single pagination sequence so that, absent concurrent changes, each matching service appears on exactly one page. A cursor MAY expire. A request with an expired or otherwise invalid `cursor` MUST be rejected with an HTTP 400 (Bad Request) ({{error-response}}); the client recovers by restarting from the first page.

The following example requests the next page using a cursor from a previous response:

    GET /.well-known/service-catalog?cursor=b2Zmc2V0OjUw HTTP/1.1
    Host: catalog.example.com
    Authorization: Bearer SlAV32hkKG...ACCESSTOKEN...
    Accept: application/json

## Catalog Response {#catalog-response}

If the request is valid and authenticated, the Catalog Provider returns an HTTP 200 (OK) response whose body is a JSON {{RFC8259}} object, the **catalog object** ({{catalog-object}}). The `Content-Type` of the response MUST be `application/service-catalog+json` ({{iana-media-type}}); the media type suffix `+json` allows generic JSON tooling to process it. A JSON Schema for the catalog object is published with this specification, and the catalog object MAY carry a `$schema` member referencing it, so that clients can validate responses and generate types. A complete example appears in {{response-example}}.

The Catalog Provider MUST evaluate the authenticated user's permissions, and the calling client's permissions, when constructing the catalog. The catalog MUST contain only services, and connection methods, that the user and client are permitted to use. The specific authorization policy evaluation is implementation specific. When constructing the response, the Catalog Provider MUST omit any member that would otherwise contain an empty string, an empty array, or a null value.

The Catalog Provider MAY include HTTP caching headers as specified in {{RFC9111}}. Because the catalog is user specific, any cache directives MUST mark the response as private (for example, `Cache-Control: private`). To let a client refresh a large per-user catalog cheaply, the Catalog Provider SHOULD support conditional requests by returning an `ETag` (and/or `Last-Modified`) and honoring `If-None-Match` (and/or `If-Modified-Since`) {{RFC9110}}, responding 304 (Not Modified) when the catalog is unchanged.

### Service Availability and Status {#availability}

"Available to the user" is split across two levels: whether a service *appears* at all, and the per-connection `status` ({{connection-object}}) of how it can be used.

A service appears in the catalog when the user is permitted to know it exists -- it is *discoverable*. Appearing does not by itself mean the user can call it now; that is conveyed per connection:

* Discoverable: the service is listed at all. A Catalog Provider MUST omit services the user is not permitted to discover.
* Connectable without interaction: at least one connection has `status` `available` or `connected`.
* Connectable after consent: a connection has `status` `consent_required`.
* Known but currently unavailable: every connection has `status` `unavailable` -- for example, the service is licensed but not provisioned for this user, or is temporarily blocked by policy. The service is still shown so the user or agent knows it exists and how it might be enabled (for example, via a `sign-up` link).

In short, inclusion answers "may the user know about this service," and `status` answers "can the user connect, and how."

### Error Response {#error-response}

If the request fails, the Catalog Provider returns an HTTP error response. The response body, when present, is a problem details object {{RFC9457}} with the media type `application/problem+json`. The HTTP status code is set as follows:

* 401 (Unauthorized): authentication is required and was missing or invalid. The Catalog Provider MUST include a `WWW-Authenticate` header field {{RFC6750}} indicating how to authenticate.
* 403 (Forbidden): the authenticated client is not authorized to use the catalog endpoint.
* 400 (Bad Request): the request is malformed, or includes an invalid or expired `cursor` (see {{pagination}}).
* 503 (Service Unavailable): the Catalog Provider is temporarily unable to handle the request.

The following is an example error response:

    HTTP/1.1 401 Unauthorized
    Content-Type: application/problem+json
    WWW-Authenticate: Bearer

    {
      "type": "https://example.com/probs/invalid-token",
      "title": "The access token is missing or expired",
      "status": 401
    }

# Catalog Data Model

This section defines the JSON objects returned by the endpoint: the catalog object and the service, link, and connection objects it contains, together with the service-type, category, and connection-type vocabularies. Worked examples follow in {{response-example}}.

## Catalog Object {#catalog-object}

The catalog object contains the following members:

$schema:
: OPTIONAL. A URI referencing the JSON Schema that describes this catalog object, for validation and type generation. This identifies a validation schema, not the protocol version.

version:
: OPTIONAL. A string giving the version of the catalog format used in this response (for example, `"1"`). A client uses it to adapt to format evolution; absent any value, the client assumes the version defined by this document.

services:
: REQUIRED. An array of **service objects** (see {{service-object}}), each describing a service available to the user. If the user has access to no services, this member is an empty array.

name:
: OPTIONAL. A string giving a human-readable name for the catalog.

description:
: OPTIONAL. A string giving a human-readable description of the catalog.

updated:
: OPTIONAL. The timestamp at which the catalog content was last updated, as an {{RFC3339}} date-time string.

next_cursor:
: OPTIONAL. An opaque string. If present, more services are available than were returned; the client obtains the next page by repeating the request with the `cursor` query parameter set to this value (see {{pagination}}). If absent, no further services are available.

Extensions MAY define additional members of the catalog object. Clients MUST ignore members they do not understand.

## Service Object {#service-object}

A service object describes a single service and how to connect to it. It contains the following members:

id:
: REQUIRED. A string giving a stable, provider-assigned identifier for the service, unique within the catalog. Clients MAY use this value to correlate a service across catalog requests.

name:
: REQUIRED. A string giving a human-readable name for the service, suitable for display to an end user (for example, in a service picker).

connections:
: REQUIRED. A non-empty array of **connection objects** (see {{connection-object}}), each describing one way the client can obtain credentials to call the service. The Catalog Provider SHOULD order the array from most to least preferred.

type:
: OPTIONAL. A string giving the service type, a value from the registry in {{service-types}}. If omitted, the type is `http`.

description:
: OPTIONAL. A string giving a human-readable description of the service.

categories:
: OPTIONAL. An array of well-known service category values (see {{service-categories}}) describing what the service does, enabling capability-based discovery.

tags:
: OPTIONAL. An array of free-form tag strings categorizing the service.

links:
: OPTIONAL. An array of **link objects** (see {{link-object}}) pointing to related resources such as documentation, sign-up, or the MCP Server Card, using link relation types {{RFC8288}}.

base_uri:
: OPTIONAL. A URI giving the primary endpoint at which the service is hosted: for an `http` service, the base URL of its API; for an `mcp` service, the MCP server endpoint. This is the network location of the service and is independent of how the client authenticates to it. It is the canonical endpoint value; a `service` link ({{link-object}}), if also present, points to the same location.

mcp:
: OPTIONAL. A JSON object present only when `type` is `mcp`; see {{type-mcp}}.

a2a:
: OPTIONAL. A JSON object present only when `type` is `a2a`; see {{type-a2a}}.

tenant:
: OPTIONAL. A string giving a machine-readable identifier for the tenant of a multi-tenant service, when this service object represents one tenant. It is descriptive -- it lets a client correlate the service, and the tokens it will obtain, with a tenant context -- and aligns with the tenant identifier used in {{TOKEN-EXCHANGE-DISCOVERY}} and {{I-D.oauth-identity-assertion-authz-grant}}.

group:
: OPTIONAL. A string giving a stable identifier shared by all service objects that are instances or tenants of the same logical service, so a client can group them (for example, in a picker). Service objects in the same group are otherwise independent: each has its own `id`, `base_uri`, `connections`, and per-user state.

group_name:
: OPTIONAL. A string giving a human-readable name for the `group`, suitable as a heading when a client presents the grouped instances or tenants together. Service objects sharing a `group` SHOULD carry the same `group_name`.

The service object intentionally carries no authentication-scheme-specific members. Values such as the authorization server, scopes, and resource indicator are properties of a particular authentication scheme, so they appear on the relevant connection object ({{connection-object}}) rather than on the service. A service that supports more than one authentication scheme has more than one connection object, each carrying the values for its scheme.

A logical service the user can reach as more than one instance or tenant -- for example, regional deployments, or `dev` and `staging` tenants of one provider -- is represented as multiple service objects that share a `group` value, each independently connectable and, where applicable, carrying its own `tenant`. This mirrors the per-tenant target representation of {{TOKEN-EXCHANGE-DISCOVERY}}: the descriptive tenant identity lives on the service object, while any acquisition-time selector (such as a token-exchange `audience`) lives on the connection.

Extensions and service types MAY define additional members of the service object. Clients MUST ignore members they do not understand.

## Service Types {#service-types}

The `type` member of a service object identifies the kind of service, using a value from the "Service Catalog Service Type" registry ({{iana-service-type}}). This document defines three values.

### http {#type-http}

A conventional HTTP API. The `base_uri` member is the base URL of the API. A machine-readable description (for example, an OpenAPI document) MAY be referenced with a `service-desc` link ({{link-object}}). This is the default type when `type` is omitted.

### mcp {#type-mcp}

A Model Context Protocol server {{MCP-AUTHORIZATION}}. The `base_uri` member is the MCP server endpoint. A service of type `mcp` SHOULD reference the server's MCP Server Card {{MCP-SERVER-CARD}}, either with an `mcp-server-card` link ({{link-object}}) or with the `server_card_uri` member below, so that a client can learn the server's capabilities without connecting.

A service of type `mcp` MAY include an `mcp` member, a JSON object with the following members:

transport:
: OPTIONAL. A string giving the MCP transport the endpoint uses (for example, `streamable-http`).

server_card_uri:
: OPTIONAL. A URI giving the location of the server's MCP Server Card {{MCP-SERVER-CARD}}.

Because MCP servers use OAuth 2.0 for authorization {{MCP-AUTHORIZATION}}, an `mcp` service typically offers OAuth-based connection methods (for example, `authorization_code` or `token_exchange`); service type and connection type are independent. As for any OAuth service, the client re-anchors to the server's Protected Resource Metadata before acting ({{connection-object}}). The location and format of the MCP Server Card are defined by {{MCP-SERVER-CARD}}; this document does not define them, and a Catalog Provider references the card by URL.

### a2a {#type-a2a}

An Agent2Agent (A2A) agent {{A2A}}. The `base_uri` member is the A2A agent endpoint. A service of type `a2a` SHOULD reference the agent's A2A Agent Card, either with an `agent-card` link ({{link-object}}) or with the `agent_card_uri` member below, so that a client can learn the agent's skills and capabilities without connecting.

A service of type `a2a` MAY include an `a2a` member, a JSON object with the following members:

transport:
: OPTIONAL. A string giving the A2A transport the endpoint uses (for example, `JSONRPC`).

agent_card_uri:
: OPTIONAL. A URI giving the location of the agent's A2A Agent Card (typically at `/.well-known/agent-card.json`).

As with `mcp`, service type and connection type are independent; an `a2a` agent describes how callers authenticate to it in its Agent Card, which the catalog references rather than restates.

## Service Categories {#service-categories}

The `categories` member of a service object, and the `category` query parameter ({{catalog-request}}), use well-known category values from the "Service Catalog Service Category" registry ({{iana-category}}) to describe what a service does. This document seeds the registry with the values `email`, `calendar`, `contacts`, `files`, `chat`, `tasks`, `crm`, and `ticketing`. The registry is extensible under a Specification Required policy ({{iana-category}}) so the shared vocabulary stays coherent; private or experimental categories use a collision-resistant namespaced form. An agent can rely on registered values to find a service by capability (for example, "an email service") rather than by name.

## Link Object {#link-object}

A link object points to a resource related to a service. It is modeled on the Web Linking framework {{RFC8288}} and contains the following members:

rel:
: REQUIRED. A link relation type {{RFC8288}}: either a registered relation type or an extension relation type (an absolute URI). Relation types commonly used in a catalog include `service` (the service's primary endpoint or home), `service-desc` (a machine-readable API description such as an OpenAPI document), `service-doc` (human-readable documentation), `terms-of-service`, `privacy-policy`, `help`, `status` (a service status page), `icon`, `describedby` (for example, a Protected Resource Metadata document {{RFC9728}}), `sign-up` (where a user can sign up for the service), `mcp-server-card` (the service's MCP Server Card {{MCP-SERVER-CARD}}), and `agent-card` (the service's A2A Agent Card {{A2A}}). The relation types `sign-up`, `mcp-server-card`, and `agent-card` are registered by this document (see {{iana-link-relations}}).

href:
: REQUIRED. A URI giving the target of the link.

type:
: OPTIONAL. A string hinting the media type of the link target.

title:
: OPTIONAL. A string giving a human-readable title for the link.

## Connection Object {#connection-object}

A connection object describes one connection method for a service, separating two layers: how the client *acquires* a credential and how it *presents* that credential when calling the service. A connection object MUST carry enough information for the client either to execute the connection directly or to deterministically fetch the remaining information it needs. Heavy or detailed metadata (for example, the full authorization server metadata document) is referenced by URL rather than inlined.

Every connection object has a `type`:

type:
: REQUIRED. A string giving the credential acquisition method, drawn from the "Service Catalog Connection Type" registry ({{connection-type-registry}}). This document defines `token_exchange`, `authorization_code`, `client_credentials`, `id_jag`, `pre_authorized`, and `none`.

**Acquisition members.** The following members carry the OAuth 2.0 values for connection methods that obtain a token from an OAuth 2.0 authorization server (`token_exchange`, `authorization_code`, `client_credentials`, and `id_jag`). They are carried on the connection object, rather than inherited from the service object or other catalog entries, so each connection is interpreted on its own.

authorization_server:
: REQUIRED for the OAuth-based connection types. A URI giving the issuer identifier {{RFC8414}} of the authorization server to use for this connection. The client obtains the authorization server's endpoints and capabilities from its metadata {{RFC8414}}.

token_endpoint:
: OPTIONAL. A URI giving the authorization server's token endpoint, provided as an optimization. Before using it, the client MUST verify it matches the `token_endpoint` in the validated metadata {{RFC8414}} of the connection's `authorization_server`; the client MUST NOT send a credential (such as a token exchange `subject_token`) to a `token_endpoint` it has not verified against that metadata. When absent, the client obtains the token endpoint from the authorization server metadata.

resource:
: OPTIONAL. A URI giving the resource indicator {{RFC8707}} the client includes when requesting a token for this connection, as a canonical URI per {{Section 2 of RFC8707}}. This identifies the protected resource and corresponds to the resource described by a Protected Resource Metadata document {{RFC9728}}, which MAY be referenced by a `describedby` link ({{link-object}}).

scopes:
: OPTIONAL. An array of OAuth 2.0 scope values {{Section 3.3 of RFC6749}} available to this connection. This is a coarse, planning-time hint and an upper bound: it lists scopes the user and client may request, not what any particular operation requires. Operation-level scope requirements come from the service's descriptor (for example, an OpenAPI {{OPENAPI}} document) and are enforced by the authorization server. A client SHOULD request the least privilege its task requires rather than the full set.

authorization_details_types:
: OPTIONAL. An array of OAuth 2.0 Rich Authorization Requests {{RFC9396}} `authorization_details` type identifiers (strings) that this connection accepts for this user, enabling fine-grained, intent-scoped access requests independently of `scopes`. Like `scopes`, this is a planning-time hint and an upper bound, not a per-operation requirement. It is the per-service, per-user counterpart to the server-wide `authorization_details_types_supported` metadata of {{RFC9396}}: it tells the client which types are usable here, while the schema, documentation, and examples for each type are obtained from the authorization server's Authorization Details Types Metadata {{RAR-METADATA}} (discovered via this connection's `authorization_server`) rather than restated in the catalog.

client_id:
: OPTIONAL. A string giving a static OAuth 2.0 client identifier the client uses with the `authorization_server`. This supports authorization servers that require a pre-registered, per-service, or per-tenant client registration.

client_registration:
: OPTIONAL. A string describing how the client obtains a client identifier when `client_id` is absent: `dynamic` (register using OAuth 2.0 Dynamic Client Registration {{RFC7591}}, discovering the registration endpoint from the authorization server metadata) or `none` (no client identifier is required). When both `client_id` and `client_registration` are absent, the client uses a client identity it determines is appropriate by other means.

**Presentation members.** The following members describe how the acquired credential is presented when calling the service.

present:
: OPTIONAL. A JSON object describing how the client presents its credential (the service-authentication layer, distinct from the acquisition `type`). Its value is an OpenAPI Security Scheme Object {{OPENAPI}} -- the same model used by OpenAPI `securitySchemes` and A2A Agent Cards {{A2A}}; because it is a verbatim OpenAPI object, its member names follow OpenAPI conventions (for example, `mutualTLS`) rather than the snake_case used elsewhere in this document. The following apply:

    * Allowed types: the value MUST be of OpenAPI type `http`, `apiKey`, or `mutualTLS` -- for example `{"type": "http", "scheme": "bearer"}`, `{"type": "apiKey", "in": "header", "name": "X-API-Key"}`, or `{"type": "mutualTLS"}`. The `oauth2` and `openIdConnect` types MUST NOT appear, because credential acquisition (including OAuth flows) is expressed by the `type` member.
    * When omitted: for acquisition types that yield an access token (`token_exchange`, `authorization_code`, `client_credentials`, `id_jag`) the credential is presented as an HTTP bearer token {{RFC6750}}; otherwise the client determines presentation from the service's referenced security schemes (see `security_scheme`).
    * No restatement: the catalog SHOULD NOT inline a `present` value that merely repeats security schemes already published in the service's descriptor (an OpenAPI document via `service-desc`, an A2A Agent Card, or an MCP Server Card); a client obtains those by reference ({{intent}}).
    * Applicability: `present` is primarily for `http` services that lack a referenced descriptor. For `mcp` services, presentation follows the MCP specification (a bearer token over the MCP transport) and neither `present` nor `security_scheme` is used; for `a2a` services, presentation follows the agent's Agent Card.
    * Sender-constrained tokens: when the resource's Protected Resource Metadata {{RFC9728}} indicates DPoP-bound access tokens are required (`dpop_bound_access_tokens_required`), or the authorization server signals DPoP support, the client MUST present a DPoP-bound token {{RFC9449}} rather than a plain bearer token. This requirement is discovered from that metadata rather than restated in the catalog.

security_scheme:
: OPTIONAL. A string naming a security scheme defined in the service's referenced descriptor (a key of an OpenAPI `securitySchemes` object, or the `securitySchemes` of an A2A Agent Card) that this connection corresponds to. This lets a client map the connection to a specific scheme rather than inferring it, and is meaningful only for the common case of a single applicable scheme; when the descriptor's security requirements are more complex (multiple required schemes, or alternatives), the client consults the descriptor's security requirements directly. This member does not apply to `mcp` services, whose authorization is defined by the MCP specification rather than by named security schemes. Presentation is resolved in this order: an explicit `present`; otherwise the named `security_scheme` resolved against the descriptor; otherwise, for OAuth-based acquisition types, an HTTP bearer token {{RFC6750}}.

**Per-user state.** The following members convey this connection's state for the authenticated user.

status:
: OPTIONAL. A string giving the per-user state of this connection method. If omitted, the client SHOULD treat the status as `available`. One of:

    * `connected`: The Catalog Provider knows of an existing user-service relationship or credential grant (for example, a prior authorization, an account link, or a stored token). This indicates a relationship exists, not that a valid credential is in hand: the client may still need to obtain or refresh a token, and the relationship MAY be stale (expired or revoked), so a client MUST still handle an authentication failure at call time.
    * `available`: The client can obtain a credential using this method without further user interaction (for example, a token exchange or client credentials grant).
    * `consent_required`: Obtaining a credential requires user interaction (for example, an authorization code grant with user consent).
    * `unavailable`: The method is described for completeness but cannot currently be used by this user.

account:
: OPTIONAL. A JSON object identifying the account this connection is associated with, when the user holds (or can establish) more than one account at the service. An account is a distinct identity or subscription the user holds at the service, established through prior authorization or account linking; how accounts are established or linked is out of scope. The object has a REQUIRED `id` (an opaque, stable, provider-assigned identifier for the account), an OPTIONAL `label` (a human-readable name for display, such as "Work"; it MAY be personal data, see {{privacy-considerations}}), and an OPTIONAL `login_hint` (a value the client passes as the OpenID Connect `login_hint` parameter to steer an interactive authorization to this account).

    Two connection objects of the same `type` that differ only by `account` represent two distinct accounts -- for example, two `connected` mailboxes. A connection with no `account` represents establishing a new account, or an account the provider does not distinguish.

    The account is honored differently per acquisition type. For interactive acquisition (`authorization_code`), the client SHOULD pass the account's `login_hint`, or otherwise rely on the authorization server's account selection, to obtain a token for that account. For silent acquisition (`token_exchange`, `id_jag`), the account is determined by the identity of the subject token (or assertion) the client presents; selecting a `connected` account therefore means using the subject token that corresponds to it. The opaque `id` is a correlation handle, not a token request parameter.

Connection types define additional type-specific members, as described in {{connection-types}}. Extensions MAY define additional members of the connection object. Clients MUST ignore members they do not understand.

### Authoritative Sources and Re-Anchoring {#authoritative}

The OAuth values in a connection object (`authorization_server`, `resource`, `scopes`, and `authorization_details_types`) are an optimization: a cache of values whose authoritative sources are the service's Protected Resource Metadata {{RFC9728}}, the authorization server metadata {{RFC8414}}, and the service's descriptor. They let a client act without extra round trips, but they are not a second source of truth. Specifically:

* Before using a connection to obtain or present a token, a client MUST re-anchor trust to the resource it intends to call: it MUST confirm that the connection's `authorization_server` is listed in the Protected Resource Metadata {{RFC9728}} of the service's `resource` (for `mcp` services, through the standard MCP flow; see {{type-mcp}}). This prevents a misconfigured or malicious catalog from directing the client to an attacker-controlled authorization server (see {{security-considerations}}).

* If a connection value conflicts with the authoritative metadata, the authoritative metadata takes precedence.

* To detect change cheaply, a client SHOULD use conditional requests against the catalog (see {{catalog-response}}); the catalog's `updated` time and entity tag indicate whether re-fetching is necessary.

The catalog's irreducible contribution is the user-scoped enumeration of which services and connection methods exist, and their per-user `status` -- not the OAuth endpoints themselves, which remain owned by the authoritative metadata.

## Connection Types {#connection-types}

This section defines the initial credential acquisition types -- the values of a connection's `type`. How the resulting credential is presented to the service is a separate layer, given by the connection's `present` member ({{connection-object}}). All reuse the common connection object members from {{connection-object}}; the members below are type specific.

### token_exchange {#type-token-exchange}

The client obtains a token by performing an OAuth 2.0 Token Exchange {{RFC8693}} at the `authorization_server`. As the `subject_token` the client presents a token it already holds whose subject is the user -- typically the access token it used to retrieve the catalog, or another user token the `authorization_server` is configured to accept for exchange. Whether this connection is truly silent (`status` of `available`) depends on the client possessing such an acceptable subject token; if it does not, the exchange will fail and the client must obtain a suitable token first. This is the catalog equivalent of {{TOKEN-EXCHANGE-DISCOVERY}}.

Type-specific members:

audience:
: OPTIONAL. A string giving the `audience` value {{Section 2.1 of RFC8693}} to include in the token exchange request. The `audience` is the logical name of the target service and need not be a URI; it MAY be an opaque, authorization-server-local identifier. For a multi-tenant service, each tenant is a distinct service object (grouped by `group`; see {{service-object}}) whose connection carries a distinct `audience` value selecting that tenant; the tenant's descriptive identity is the service object's `tenant` member.

supported_token_types:
: OPTIONAL. An array of token type URIs {{RFC8693}} that may be requested. If omitted, the client may request any token type the authorization server supports.

### authorization_code {#type-authorization-code}

The client obtains a token using the OAuth 2.0 authorization code grant {{Section 4.1 of RFC6749}} with PKCE. This type typically has `status` of `consent_required`. The client uses the `authorization_server` metadata {{RFC8414}} to locate endpoints, includes the `resource` parameter {{RFC8707}}, and uses `client_id` or `client_registration` to determine its client identity. This type defines no additional members.

### client_credentials {#type-client-credentials}

The client obtains a token using the OAuth 2.0 client credentials grant {{Section 4.4 of RFC6749}}, authenticating as itself. The resulting token carries no user identity; this type is appropriate only when the agent acts as itself and the service does not need the user's identity, and the catalog includes it because such a service may still be relevant to the user's task. Its `status` reflects the client's own access, not the user's. This type defines no additional members.

### id_jag {#type-id-jag}

The client obtains an access token by redeeming an Identity Assertion Authorization Grant (ID-JAG) {{I-D.oauth-identity-assertion-authz-grant}} at the service's `authorization_server`. The client first obtains an ID-JAG for the user from the user's identity provider and then redeems it.

Type-specific members:

audience:
: OPTIONAL. A string giving the audience value identifying the service for which the ID-JAG is requested. For a multi-tenant service, the tenant is represented as in {{type-token-exchange}}: a distinct, grouped service object carrying the descriptive `tenant`, with the audience selecting it.

### none {#type-none}

The service requires no client authentication (it is public). This type defines no additional members and typically has `status` of `available`.

### pre_authorized {#type-pre-authorized}

The credential is one the client already possesses, or can obtain through a deployment-specific secure channel (for example, a secrets manager, an enterprise configuration system, or prior provisioning) -- never through the catalog. The catalog only signals that this method applies and, via the connection's `present` member ({{connection-object}}), how the credential is presented; it MUST NOT include the credential itself or any pointer that would let an unauthorized party retrieve it. This type typically has `status` of `connected`.

A Catalog Provider MUST NOT include secret values (access tokens, refresh tokens, client secrets, API keys, or private keys) anywhere in the catalog.

## Examples {#response-example}

### A Catalog Response

The following is a non-normative example response showing four services: an HTTP service in the `email` category offering both token exchange and user consent, an MCP server referencing its Server Card, a service the user is already connected to via a pre-provisioned API key (note the `present` member), and an A2A agent referencing its Agent Card.

    HTTP/1.1 200 OK
    Content-Type: application/service-catalog+json
    Cache-Control: private, max-age=60

    {
      "$schema": "https://example.com/schemas/service-catalog.json",
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
             "href": "https://mcp.example.com/server-card.json"}
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
          "links": [
            {"rel": "service-desc",
             "href": "https://api.calendar.example/openapi.json"}
          ],
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

### Multiple Instances and Accounts {#instances-accounts}

A logical service the user can reach as more than one instance or tenant is returned as multiple service objects sharing a `group`; the user's multiple accounts at a service are returned as multiple connection objects distinguished by `account`. The following non-normative fragment shows the `services` array of such a catalog: two tenants of one logical service, and a service the user is connected to under two accounts (with a third connection to add another).

    "services": [
      {
        "id": "saas-dev",
        "name": "SaaS Example (Dev)",
        "group": "saas-example",
        "group_name": "SaaS Example",
        "tenant": "dev",
        "base_uri": "https://api.saas.example",
        "connections": [
          {
            "type": "token_exchange",
            "status": "available",
            "authorization_server": "https://as.saas.example",
            "resource": "https://api.saas.example",
            "audience": "urn:saas:tenant:dev"
          }
        ]
      },
      {
        "id": "saas-staging",
        "name": "SaaS Example (Staging)",
        "group": "saas-example",
        "group_name": "SaaS Example",
        "tenant": "staging",
        "base_uri": "https://api.saas.example",
        "connections": [
          {
            "type": "token_exchange",
            "status": "available",
            "authorization_server": "https://as.saas.example",
            "resource": "https://api.saas.example",
            "audience": "urn:saas:tenant:staging"
          }
        ]
      },
      {
        "id": "mail",
        "name": "Example Mail",
        "categories": ["email"],
        "base_uri": "https://api.example.com/mail",
        "connections": [
          {
            "type": "token_exchange",
            "status": "connected",
            "account": {"id": "acct-1", "label": "Work"},
            "authorization_server": "https://as.example.com",
            "resource": "https://api.example.com/mail"
          },
          {
            "type": "token_exchange",
            "status": "connected",
            "account": {"id": "acct-2", "label": "Personal"},
            "authorization_server": "https://as.example.com",
            "resource": "https://api.example.com/mail"
          },
          {
            "type": "authorization_code",
            "status": "consent_required",
            "authorization_server": "https://as.example.com",
            "resource": "https://api.example.com/mail",
            "client_registration": "dynamic"
          }
        ]
      }
    ]

# Connecting to a Service {#connecting}

After retrieving the catalog, a client connects to a service by selecting a service object and one of its connection objects, and then executing the corresponding connection type. For planning, the connection's `status` is the primary signal -- whether a credential already exists (`connected`), can be obtained without user interaction (`available`), or requires consent (`consent_required`) -- while the acquisition `type` is the mechanism detail the client uses once a connection is chosen. The general procedure is:

1. Select a service (for example, by `category`, `tags`, or by presenting `name` and `description` to the user). For an `mcp` service, the client MAY first fetch the MCP Server Card ({{type-mcp}}) to evaluate the server's capabilities.
2. Select a connection object. A client SHOULD prefer a connection whose `status` is `connected` or `available` over one whose `status` is `consent_required`, when more than one is suitable.
3. For OAuth-based connection types, determine the client identity: use `client_id` if present; otherwise, if `client_registration` is `dynamic`, register with the authorization server using {{RFC7591}} -- but only after applying a trust policy to the catalog-discovered authorization server ({{confused-deputy}}); otherwise proceed without a client identifier if `client_registration` is `none`.
4. Obtain the credential according to the connection `type` (see {{connection-types}}), using the values on the selected connection object, including the `resource` {{RFC8707}} and `scopes` for OAuth-based types, after re-anchoring to the resource's metadata ({{authoritative}}).
5. Call the service, presenting the credential as described by the connection's `present` member or, when it is absent, by the service's referenced security schemes (an OpenAPI {{OPENAPI}} document, an A2A Agent Card, or an MCP Server Card) -- for example, as a bearer token {{RFC6750}}, an API key, or a client certificate.

This specification does not define any new credential issuance mechanism. Each connection type parameterizes an existing mechanism.

# Intent-Based Use and Resource Discovery {#intent}

An agent typically uses the catalog to plan: it discovers the services a user can reach, decides which it needs for the user's goal, and requests only the access that goal requires. The catalog supports planning before commitment in two ways.

First, a client can plan from descriptors without connecting:

* A service object's `links` reference the service's capability descriptor -- for example, an OpenAPI {{OPENAPI}} document via a `service-desc` link, or an MCP Server Card {{MCP-SERVER-CARD}} via an `mcp-server-card` link. A client MAY read these to understand a service's operations and resources, and plan, without obtaining any token.

* The connection's `scopes` and `authorization_details_types` are only an upper bound ({{connection-object}}); the authoritative, fine-grained requirements come from the service, and their granularity differs by service type. An `http` service described by OpenAPI exposes operations with per-operation `security` and required scopes; an `mcp` service exposes tools enumerated at runtime, where scopes are typically server-wide (via the server's Protected Resource Metadata {{RFC9728}}) rather than per-tool; an `a2a` service exposes skills described in its Agent Card. A client maps its intended operations, tools, or skills to the minimal scopes and authorization details it must request.

* For a multi-step goal spanning operations or services, a client MAY use a workflow description such as Arazzo {{ARAZZO}} to plan the sequence before acquiring any token.

Second, a client can connect with minimal access to enumerate, then request what is needed. When an agent must inspect the user's actual resources before planning (for example, to list the user's mailboxes or calendars), it SHOULD select a connection and request the least-privilege access sufficient to enumerate -- narrow `scopes`, or a minimal `authorization_details` request limited to the `authorization_details_types` the connection advertises ({{connection-object}}) -- and then make a further, intent-scoped request for the access it determines it needs.

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

Several of the above are evolving specifications: A2A {{A2A}}, MCP Server Cards {{MCP-SERVER-CARD}}, and {{RAR-METADATA}} are referenced here as directional rather than stable dependencies. The catalog's core function -- per-user enumeration of services and their connection methods -- does not depend on any of them; a Catalog Provider and client can interoperate using only OAuth 2.0 metadata ({{RFC9728}}, {{RFC8414}}) and human-readable links, and treat references to those evolving formats as optional enhancements.

# Security Considerations {#security-considerations}

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

A client trusts the Catalog Provider to enumerate services, endpoints, and connection methods accurately. Because the catalog aggregates connection information for potentially all of a user's services, a compromised or overly broad Catalog Provider is a high-value single point that could direct the client to attacker-controlled services or authorization servers for many services at once. An autonomous client typically has no site-specific policy to judge an endpoint; it therefore MUST re-anchor trust to the resource being called -- confirming, via the resource's Protected Resource Metadata {{RFC9728}}, that the connection's `authorization_server` is authoritative for that `resource` ({{authoritative}}) -- before obtaining or presenting any credential. A client SHOULD obtain the Catalog Provider's base URL from a trusted source ({{endpoint-discovery}}) and SHOULD apply the same scrutiny to resources referenced by `links` (such as a `service-desc` or `mcp-server-card`).

## Confused Deputy and Client Registration {#confused-deputy}

When `client_registration` is `dynamic` ({{RFC7591}}), a client registers with authorization servers it may not have known in advance. A client MUST apply appropriate consent and validation before registering with, or obtaining tokens from, an authorization server discovered through the catalog, consistent with the confused-deputy guidance in {{MCP-AUTHORIZATION}}. Before dynamically registering with a catalog-discovered authorization server, a client SHOULD gate the registration on a trust policy -- an allowlist of acceptable issuers, membership in a trust framework, an issuer-matching policy, or explicit user or administrator confirmation -- rather than registering automatically.

## Authorization Server Mix-Up

A catalog directs a client to potentially many authorization servers, and an agent may run several authorization flows concurrently -- the conditions under which authorization server mix-up attacks arise. A client using the `authorization_code` connection type SHOULD use the `iss` authorization response parameter {{RFC9207}} and verify it identifies the expected authorization server, and MUST otherwise follow the mix-up mitigations of current OAuth security best practice. Re-anchoring each connection's `authorization_server` to the resource's Protected Resource Metadata ({{authoritative}}) further reduces this risk.

# Privacy Considerations {#privacy-considerations}

The catalog is inherently personal: it describes the services a specific user can access. This may reveal organizational affiliations, tenant memberships, and the user's relationships to third-party services. To protect privacy:

* The catalog, and any per-user catalog endpoint URL, MUST be obtained through an authenticated channel ({{endpoint-discovery}}). In particular, WebFinger {{RFC7033}} MUST NOT be used to convey the catalog endpoint or its contents: WebFinger responses are unauthenticated and, by design, broadly accessible (including cross-origin), so publishing the set of services a user can reach there would disclose personal data to anyone. WebFinger is used only to resolve a user identifier to an issuer.
* The Catalog Provider SHOULD return only information the authenticated user and client are authorized to know, applying the principle of least privilege.
* The Catalog Provider SHOULD NOT include identifiers of other users or of tenants the user is not associated with.
* Account labels (`account.label`) MAY be personal data (for example, an email address). The Catalog Provider SHOULD return the minimum needed for the user to distinguish accounts -- a display name or masked address -- and MAY omit the label entirely, leaving only the opaque `account.id`.
* The Catalog Provider SHOULD log access in accordance with applicable privacy regulations and SHOULD minimize retention of catalog requests.
* A deployment MAY provide mechanisms for users to review and limit the services exposed through the catalog.

# IANA Considerations

## OAuth Authorization Server Metadata {#iana-as-metadata}

IANA is requested to register the following value in the IANA "OAuth Authorization Server Metadata" registry established by {{RFC8414}}.

Metadata Name: `service_catalog_endpoint`

Metadata Description: URL of the per-user Service Catalog Endpoint associated with this authorization server.

Change Controller: IETF

Specification Document(s): [[ this document ]]

## Media Type {#iana-media-type}

IANA is requested to register the following media type in the "Media Types" registry, following the procedure of {{RFC6838}}.

Type name: application

Subtype name: service-catalog+json

Required parameters: N/A

Optional parameters: N/A

Encoding considerations: binary; the content is a JSON {{RFC8259}} value encoded in UTF-8.

Security considerations: See {{security-considerations}} of this document.

Interoperability considerations: N/A

Published specification: [[ this document ]]

Applications that use this media type: OAuth 2.0 clients and autonomous agents that retrieve a per-user service catalog.

Fragment identifier considerations: Follows the `+json` structured syntax suffix semantics.

Change controller: IETF

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

IANA is requested to establish the "Service Catalog Service Category" registry for values of the `categories` member of a service object and the `category` query parameter ({{service-categories}}). The registration policy is Specification Required, so that the well-known category vocabulary agents plan against stays coherent. Each registration contains the category value, a brief description, and a reference. Values are case-sensitive strings of printable ASCII characters without whitespace. An implementation-specific or experimental category that is not registered MUST use a collision-resistant, namespaced form (for example, a reverse-DNS prefix such as `com.example.crm`) so it cannot clash with a registered value. Initial contents:

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
