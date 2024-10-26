---
stand_alone: true
ipr: trust200902
cat: info # Check
submissiontype: IETF

docname: draft-ietf-crest-latest
venue:
  github: "SGNL-ai/crest"
  latest: ""

title: The Contextualized REST Architecture
abbrev: crest
lang: en
kw:
  - Microservices
  - REST
  - APIs
  - 

# date: -- date is filled in automatically by xml2rfc if not given
author:
- ins: A. Tulshibagwale
  name: Atul Tulshibagwale
  org: SGNL
  email: atul@sgnl.ai

normative:
  RFC4648: # Base-N Encodings

informative:
  REST:
    title: Representational State Transfer (REST)
    target: https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
    author:
    - name: Roy Thomas Fielding
      email: fielding@apache.org

  SPIFFE:
    title: Secure Production Identity Framework for Everyone
    target: https://spiffe.io/docs/latest/spiffe-about/overview/
    author:
    - org: Cloud Native Computing Foundation
  TRATS:
    title: Transaction Tokens
    target: https://datatracker.ietf.org/doc/draft-ietf-oauth-transaction-tokens/
    author:
    - name: Atul Tulshibagwale
      org: SGNL
      email: atul@sgnl.ai
    - name: George Fletcher
      org: Capital One
      email: george.fletcher@capitalone.com
    - name: Pieter Kasselman
      org: SPIRL
      email: pieter@spirl.com

--- abstract

The REST architecture is extremely popular in modern computing environments. A benefit it provides is that each request can be serviced by independent server side components such as different instances of web servers or different instances of serverless request handlers.

The lack of any context associated with a request means that the client has to provide the entire context with every request. In monolithic server-side architectures the client typically only provides an identifier to a previously established "session" on the server side, which is retrieved from a persistent storage such as a database.

However, this strategy often does not work in microservices architectures, where the persistent storage may be many hops away from the server-side components that handle the incoming REST API requests. As a result, REST APIs are used between services and each request is required to carry large context including tokens such as Transaction Tokens {{TRATS}} (which assure the identity and context of each request), SPIFFE {{SPIFFE}} tokens (which assures the service identity) and possibly other context.

The Contextualized REST (CREST) architecture adds a cached context to REST, with mechanisms to negotiate the establishment of the context when it is not found. Each request can refer to the context items it depends upon instead of carrying the entire context with the request. The server can accept the reference to the context item or respond with a specific error to prompt the client to reestablish the context. Clients can create new or delete existing context items in separate requests or as a part of other requests.

The CREST architecture assumes the server holds the context across different requests in a server-side cache. Such a cache may be typically shared across various applications and services within a VPC or a data center. The possibility that subsequent requests from the same client will reach different VPCs or data centers is low, thus providing an efficiency optimization for the vast majority of requests.

--- middle

# Introduction

The REST architecture has been very successful and used almost universally in modern computing environments. However, it requires each request to carry the entire context of the request so that the server servicing the HTTP request can process it. A benefit of this approach is that each request from the same client can be serviced by different server-side instances, thereby making it easier to load-balance and provide better availability, reliability and response-time characteristics. As the amount of data in the context grows, this strategy, however, causes each request to be more expensive in terms of the network traffic and creates latency of its own.

The Contextualized REST (CREST) architecture addresses this by enabling HTTP servers to cache independently addressed context items. One or more server-side instances can share the same cache, making it more efficient than carrying the context with each request. At the same time, the CREST architecture enables different collections of server-side instances to have different caches. This enables geographically or network separated instances to maintain different caches.

# The Context Cache
The Context Cache (CC) is an abstraction that assumes one or more server side components can access a cache which can hold context across requests.

## Cached Items
The CC is organized as a collection of independent items. Each item can be created, read, updated and deleted independently. Items may expire at any time. Each Item is identified by a unique identifier called `CCII` (Context Cache Item Identifier). The CCII format is a Base64 encoded with URL and Filename Safe Alphabet as described in RFC4648 {{RFC4648}}.

# CCII Header {#ccii-header}
When a client wants to make a request by referencing a CC Item, it includes a header named `CCII` in the request. The value of this header is a comma separated list of CCIIs.

~~~ HTTP
CCII  ccii1, ccii2, ccii3
~~~
{: #fig-ccii-header title="Non-normative example of a CCII Request Header"}

# Contextualized REST HTTP Request {#crest-request}
A contextualized REST HTTP Request includes exactly one CCII header {{ccii-header}} that references the Context Cache Items that it depends upon.

## Cache Miss
If the HTTP server is unable to find the Context Cache Item referenced in a CREST Request {{crest-request}}, it MUST return the status code `424` (Failed Dependency) to the HTTP client. The body of the HTTP response SHOULD contain the `MISSED-CCII` header. The value of this header MUST be a comma separated list of one or more items that were not found in the cache.

~~~ HTTP
MISSED-CCII ccii1, ccii2
~~~
{: #fig-missed-ccii-header title="Non-normative example of a MISSED-CCII Response Header"}

# Context Cache Operations
A HTTP client MAY create or delete a CC Item. These operations are done by including specific headers in the HTTP request and obtaining specific headers in the HTTP response.

## Create CC Items
To create a CC Item, the HTTP client includes the request header named `CC-CreateItems`. The value of the `CC-CreateItems` header is a comma separated list of URL Safe Base64 {{RFC4648}} encoding of the values to be inserted into the CC. If zero or more of the specified values already exist in the CC or if the HTTP server creates one or more of the specified values in the CC, it includes the `CCII` header in the response, and sets its value to the CCII of the item specified in the request.

The HTTP client MUST accept the CCII headers in all responses with status codes except responses that have the status codes greater than 499.

~~~ HTTP
CC-CreateItems abc123-efg456_,789hij_012,asdkflkau3
~~~
{: #fig-createitem-request title="Non-normative example of a CCII-CreateItem Request Header"}

~~~ HTTP
CCII ccii3,ccii4
~~~
{: #fig-createitem-response title="Non-normative example of a CCII Response Header"}

## Delete CC Items
To delete a CC Item, the HTTP client includes the request header named `CC-DeleteItems`. The value of the `CC-DeleteItems` header is a comma separated list of CCIIs. The HTTP Server MAY delete the requested CC Items. The HTTP Server confirms the deletion by including a `Deleted-CCII` header in the HTTP response.

~~~ HTTP
CC-DeleteItems ccii1,ccii3
~~~
{: #fig-deleteitem-request title="Non-normative example of a CCII-DeleteItem Request Header"}

~~~ HTTP
Deleted-CCII ccii1
~~~
{: #fig-deleteitem-response title="Non-normative example of a Deleted-CCII Response Header"}

# Architecture Notes
This is a non-normative section.

The CREST architecture enables a pool of HTTP server components to share a cache and a request received at any one in the pool to reference the cache. This can be achieved in today's computing environments within the limits of a VPC or a data center.

Since most client requests would route to the same data center, this strategy can provide improved efficiency over the requirement to pass the entire context with each request.

--- back

# Acknowledgements {#Acknowledgements}
{: numbered="false"}

