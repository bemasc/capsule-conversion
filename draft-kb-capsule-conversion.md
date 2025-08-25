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
 -
    fullname:
      :: 奥 一穂
      ascii: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com

normative:

informative:


--- abstract

This draft specifies how to mark Capsule Protocol requests so that HTTP intermediaries can translate them between HTTP versions.

--- middle

# Introduction

The Capsule Protocol {{!CAPSULE=RFC9297}} defines a framing layer that can be used for protocols running over HTTP.  The Capsule Protocol consists of a linear stream of capsules, each with a specified type and size.  Endpoints can establish a Capsule Protocol connection using HTTP Extended CONNECT or the HTTP/1.1 Upgrade process.

Intermediaries such as HTTP Gateways can play an active role in the Capsule Protocol.  To inform intermediaries that this protocol is in use, endpoints can include a "Capsule-Protocol: ?1" header in their request or response.  Intermediaries are obligated to pass unrecognized capsule types unmodified, but some capsule types do permit intermediaries to modify them.

The Capsule Protocol is defined for HTTP/1.1, HTTP/2, and HTTP/3, and intermediaries can translate Capsule Protocol streams between different versions of HTTP.  For example, an HTTP Gateway receiving a request using the "connect-tcp" Upgrade Token {{?I-D.ietf-httpbis-connect-tcp}} over HTTP/3 might forward it to a backend server using HTTP/1.1 for further processing.  However, this translation currently requires the intermediary to recognize the specified Upgrade Token.  Unrecognized Upgrade Tokens cannot be translated between HTTP versions.

~~~ aasvg
 .------.                .-------.                  .------.
| Client +--> HTTP/2--->| Gateway +--> HTTP/1.1--->| Server |
 '------'  :protocol=foo '-------'   Upgrade: foo   '------'
        capsule-protocol=?1        Capsule-Protocol: ?1
~~~
{: title="HTTP version translation of an Upgrade Token.  The gateway can only perform this translation if it recognizes the \"foo\" Upgrade Token."}

As a result of this limitation, HTTP intermediaries cannot forward unrecognized Capsule Protocol Upgrade Tokens (CPUTs) unless the backend supports the HTTP version used by the client.  In practice, such HTTP version mismatches are common, so intermediaries have preferred not to support unrecognized tokens at all.  As a result, each new CPUT requires the cooperation of any HTTP intermediaries.  This increases the maintenance burden on intermediaries and impedes the deployment of novel CPUTs.

This draft defines a new parameter for marking Capsule Protocol requests as safe to convert between HTTP versions, allowing intermediaries to perform such translations for unrecognized upgrade tokens.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# New Parameter: "convertible"

This specification defines a new Boolean parameter for the "Capsule-Protocol" header field, with the name "convertible".  A value of Boolean true indicates that the upgrade token is trivially convertible between HTTP versions.  An upgrade token is trivially convertible if the upgraded protocol can safely be converted between HTTP versions without any modifications beyond those required by the Capsule Protocol {{!CAPSULE}}.

The "convertible" parameter MAY appear on request header fields with a true or false Boolean item value.  If the "Capsule-Protocol" item value is Boolean false, and the "convertible" parameter value is Boolean true, this indicates that the upgrade token is trivially convertible between HTTP versions even though it does not comply with the Capsule Protocol.

The "convertible" parameter MUST NOT appear in response header fields.

## Existing Convertible Upgrade Tokens

The following pre-existing upgrade tokens are trivially convertible:

* "connect-udp" {{?CONNECT-UDP=RFC9298}}
* "connect-ip" {{?CONNECT-IP=RFC9484}}
* "connect-tcp" {{?CONNECT-TCP=I-D.ietf-httpbis-connect-tcp}}

The WebSocket protocol {{?RFC6455}}{{?RFC8441}} is not trivially convertible due to version-specific requirements for the "Sec-WebSocket-Key" and "Sec-WebSocket-Accept" header fields.

> TODO: Comment on WebTransport, pending the result of https://github.com/ietf-wg-webtrans/draft-ietf-webtrans-http3/issues/218

Future upgrade token definitions should indicate whether they are trivially convertible.

# Requirements for Intermediaries

This section describes requirements on HTTP intermediaries that change the HTTP version of a Capsule Protocol request.

## Converting an HTTP/1.1 Upgrade request to Extended CONNECT

A Convertible Upgrade Request is a request that meets these criteria:

* The HTTP version is "1.1".
* The method is "GET".
* An "Upgrade" header field is present and specifies exactly one Upgrade Token.
* The upgrade token is known to be trivially convertible, because:
   - A "Capsule-Protocol" header field is present with a Boolean true "convertible" parameter, OR
   - The intermediary recognizes this Upgrade Token as trivially convertible.
* The request is otherwise well-formed.

Upon receiving a Convertible Upgrade Request, an HTTP intermediary MAY convert it into an Extended CONNECT request ({{!RFC9220}}{{!RFC8441}}) using ordinary HTTP version translation with the following modifications:

* Change the method to "CONNECT".
* Add a ":protocol" pseudo-header containing the specified Upgrade Token.

(Note that ordinary HTTP version translation removes the "Connection" and "Upgrade" headers.)

Intermediaries MAY add a "capsule-protocol" request header and/or a "convertible" parameter if one or both are missing.

If the intermediary receives a "200 (OK)" response, it MUST convert it to an HTTP/1.1 Upgrade response as follows:

* Change the response code to "101 (Switching Protocols)".
* Add an "Upgrade" header containing the specified Upgrade Token.
* Add a "Connection: Upgrade" header.

After sending this response, the intermediary MUST process all data to and from the client in accordance with the Capsule Protocol.

If the intermediary receives any other valid response, it MUST NOT convert it to an HTTP/1.1 Upgrade response, and MUST forward it using ordinary HTTP version translation.  If the response status was not 1xx (Informational), the intermediary MAY accept additional HTTP/1.1 requests on this connection to the client.

## Converting an Extended CONNECT request to HTTP/1.1 Upgrade

A Convertible Extended CONNECT request is a request that meets these criteria:

* The method is "CONNECT".
* The ":protocol" pseudo-header is present, providing an Upgrade Token.
* The upgrade token is known to be trivially convertible, because
   - A "capsule-protocol" header field is present with a Boolean true "convertible" parameter, OR
   - The intermediary recognizes this Upgrade Token as trivially convertible.
* The request is otherwise well-formed.

Upon receiving a Convertible Extended CONNECT Request, an HTTP intermediary MAY convert it into an HTTP/1.1 Upgrade request according to ordinary HTTP version translation, with the following modifications:

* Change the method to "GET".
* Replace the ":protocol" pseudo-header with an "Upgrade" header field containing the same Upgrade Token.
* Add a "Connection: Upgrade" header.

Intermediaries MAY add a "Capsule-Protocol" request header and/or a "convertible" parameter if one or both are missing.

If the intermediary receives a correctly formed "101 (Switching Protocols)" response, it MUST change the response code to "200 (OK)".  If it receives a 2xx (Successful) response, it SHOULD return a "501 (Not Implemented)" status code, to indicate that the ":protocol" value was not accepted ({{!RFC9220, Section 3}}). Otherwise, it MUST forward any valid responses unmodified.  After sending a "200" response, the intermediary MUST process all further data to and from the server in accordance with the Capsule Protocol.

## Converting an Extended CONNECT request to Extended CONNECT for a different HTTP version

An HTTP intermediary MAY translate a Convertible Extended CONNECT Request between different HTTP versions using ordinary HTTP version translation.

# Security Considerations

When implemented incorrectly, HTTP/1.1 Upgrade carries a risk of request smuggling.  (See {{?I-D.ietf-httpbis-optimistic-upgrade}}.)

Intermediary implementors should take care to avoid excessive resource consumption by malicious clients.  For example, a client might be able to send many short requests over a single HTTP/2 or HTTP/3 connection, each of which requires opening a new, long-lived TCP connection to the backend.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
