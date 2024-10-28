---
title: "HTTP Version Translation of the Capsule Protocol"
abbrev: "Capsule Translation"
category: std

docname: draft-kb-capsule-conversion-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: WIT
# workgroup: HTTPBIS
keyword:
venue:
  github: "bemasc/capsule-conversion"

author:
 -
    fullname: Benjamin M. Schwartz
    organization: Meta Platforms, Inc.
    email: ietf@bemasc.net

normative:

informative:


--- abstract

This draft specifies how HTTP intermediaries can translate the Capsule Protocol between HTTP versions.

--- middle

# Introduction

The Capsule Protocol {{!RFC9297}} defines a framing layer that can be used for protocols running over HTTP.  The Capsule Protocol consists of a linear stream of capsules, each with a specified type and size.  Endpoints can establish a Capsule Protocol connection using HTTP Extended CONNECT or the HTTP/1.1 Upgrade process.

Intermediaries such as HTTP Gateways can play an active role in the Capsule Protocol.  To inform intermediaries that this protocol is in use, endpoints can include a "Capsule-Protocol: ?1" header in their request or response.  Intermediaries are obligated to pass unrecognized capsule types unmodified, but some capsule types do permit intermediaries to modify them.

The Capsule Protocol is defined for HTTP/1.1, HTTP/2, and HTTP/3, and intermediaries can translate Capsule Protocol streams between different versions of HTTP.  For example, an HTTP Gateway receiving a request using the "connect-tcp-capsule" Upgrade Token {{?I-D.ietf-httpbis-connect-tcp}} over HTTP/3 might forward it to a backend server using HTTP/1.1 for further processing.  However, this translation currently requires the intermediary to recognize the specified Upgrade Token.  Unrecognized Upgrade Tokens cannot be translated between HTTP versions.

~~~ aasvg
 .------.                .-------.                  .------.
| Client +--> HTTP/2--->| Gateway +--> HTTP/1.1--->| Server |
 '------'  :protocol=foo '-------'   Upgrade: foo   '------'
        capsule-protocol=?1        Capsule-Protocol: ?1
~~~
{: title="HTTP version translation of an Upgrade Token.  The gateway can only perform this translation if it recognizes the \"foo\" Upgrade Token."}

As a result of this limitation, HTTP intermediaries cannot forward unrecognized Capsule Protocol Upgrade Tokens (CPUTs) unless the backend supports the HTTP version used by the client.  In practice, such HTTP version mismatches are common, so intermediaries have preferred not to support unrecognized tokens at all.  As a result, each new CPUT requires the cooperation of any HTTP intermediaries.  This increases the maintenance burden on intermediaries and impedes the deployment of novel CPUTs.

This draft specifies general rules for translating Capsule Protocol requests across HTTP versions, allowing intermediaries to perform such translations for unrecognized CPUTs.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Requirements

This section describes requirements on HTTP intermediaries that change the HTTP version of a Capsule Protocol request.

## Converting an HTTP/1.1 Upgrade request to Extended CONNECT

A Convertible Upgrade Request is a request that meets these criteria:

* The HTTP version is "1.1".
* The method is "GET".
* The Upgrade header is present and specifies exactly one Upgrade Token.
* A "Capsule-Protocol" header is present with an item value of "?1" (with or without parameters).
* The request is otherwise well-formed for use with the Capsule Protocol.

Upon receiving a Convertible Upgrade Request, an HTTP intermediary MAY convert it into an Extended CONNECT request ({{!RFC9220}}{{!RFC8441}}) using ordinary HTTP version translation with the following modifications:

* Change the method to "CONNECT".
* Add a ":protocol" pseudo-header containing the specified Upgrade Token.

(Note that ordinary HTTP version translation removes the "Connection" and "Upgrade" headers.)

If the intermediary receives a "200 (OK)" response, it MUST convert it to an HTTP/1.1 Upgrade response as follows:

* Change the response code to "101 (Switching Protocols)".
* Add an "Upgrade" header containing the specified Upgrade Token.
* Add a "Connection: Upgrade" header.

After sending this response, the intermediary MUST process all data to and from the client in accordance with the Capsule Protocol.

If the intermediary receives any other valid response, it MUST NOT convert it to an HTTP/1.1 Upgrade response, and MUST forward it using ordinary HTTP version translation.  If the response status was not 1xx (Informational), the intermediary MAY accept additional HTTP/1.1 requests on this connection to the client.

## Converting an Extended CONNECT request to HTTP/1.1 Upgrade

A Convertible Extended CONNECT request is a request that meets these criteria:

* The method is "CONNECT".
* A "capsule-protocol" header is present with an item value of "?1" (with or without parameters).
* The request is otherwise well-formed for use with the Capsule Protocol.

Upon receiving a Convertible Extended CONNECT Request, an HTTP intermediary MAY convert it into an HTTP/1.1 Upgrade request according to ordinary HTTP version translation, with the following modifications:

* Change the method to "GET".
* Add an "Upgrade" header containing the specified Upgrade Token.
* Add a "Connection: Upgrade" header.

If the intermediary receives a correctly formed "101 (Switching Protocols)" response, it MUST change the response code to "200 (OK)".  If it receives a 2xx (Successful) response, it SHOULD return a "501 (Not Implemented)" status code, to indicate that the ":protocol" value was not accepted ({{!RFC9220, Section 3}}). Otherwise, it MUST forward any valid responses unmodified.  After sending a "200" response, the intermediary MUST process all further data to and from the server in accordance with the Capsule Protocol.

## Converting an Extended CONNECT request to Extended CONNECT for a different HTTP version

An HTTP intermediary MAY translate a Convertible Extended CONNECT Request between different HTTP versions using ordinary HTTP version translation.

## Adjusting for Server Capabilities

When translating between HTTP versions, HTTP intermediaries often need to forward requests to servers that do not support all possible HTTP versions.  This can be accomplished by explicit configuration, by a fallback mechanism, or in some other way.

An HTTP intermediary implementing this specification MUST account for backend servers that support HTTP/2 or HTTP/3, but do not implement Extended CONNECT.  When the backend server does not support Extended CONNECT, the intermediary MUST forward the request over HTTP/1.1.

# Implications

Translation of unrecognized CPUTs across HTTP versions carries some implications for future specifications related to the Capsule Protocol:

* All CPUTs must treat "GET" in HTTP/1.1 as semantically equivalent to Extended CONNECT.
* The "Capsule-Protocol" response header has no effect and should be treated as a hint for later analysis.  Intermediaries can process the response as the Capsule Protocol based entirely on the request header and the response status code.
* Intermediaries' behavior regarding each capsule type is independent of the CPUT.  CPUTs cannot change intermediaries' treatment of existing capsule types.
* A Capsule Type or CPUT cannot change the meaning of an HTTP extension.  It can only rely on the behaviors that are defined as mandatory for any implementation of that extension.  Extensions intended for use with the Capsule Protocol will likely need to define how HTTP version translation works.

All existing CPUTs and Capsule Types already conform to these rules.

# Security Considerations

When implemented incorrectly, HTTP/1.1 Upgrade carries a risk of request smuggling.  (See {{?I-D.ietf-httpbis-optimistic-upgrade}}.)

Intermediary implementors should take care to avoid excessive resource consumption by malicious clients.  For example, a client might be able to send many short requests over a single HTTP/2 or HTTP/3 connection, each of which requires opening a new, long-lived TCP connection to the backend.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
