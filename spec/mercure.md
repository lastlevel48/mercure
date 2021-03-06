%%%
title = "The Mercure Protocol"
area = "Internet"
workgroup = "Network Working Group"

[seriesInfo]
name = "Internet-Draft"
value = "draft-dunglas-mercure-05"
stream = "IETF"
status = "informational"

[[author]]
initials="K."
surname="Dunglas"
fullname="Kévin Dunglas"
abbrev = "Les-Tilleuls.coop"
organization = "Les-Tilleuls.coop"
  [author.address]
  email = "kevin@les-tilleuls.coop"
  [author.address.postal]
  city = "Lille"
  street = "82 rue Winston Churchill"
  code = "59160"
  country = "France"
%%%

.# Abstract

Mercure is a protocol enabling the pushing of data updates to web browsers and other HTTP clients in
a fast, reliable and battery-efficient way. It is especially useful for publishing real-time updates
of resources served through web APIs to web and mobile apps.

{mainmatter}

# Terminology

The keywords **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD
NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL**, when they appear in this document, are to be
interpreted as described in [@!RFC2119].

 *  Topic: The unit to which one can subscribe to changes. The topic **MUST** be identified by an
    IRI [@!RFC3987] or by a string. Using an HTTPS [@!RFC7230] or HTTP [@!RFC7230] URI [@!RFC3986]
    is **RECOMMENDED**.

 *  Publisher: An owner of a topic. Notifies the hub when the topic feed has been updated. As in
    almost all pubsub systems, the publisher is unaware of the subscribers, if any. Other pubsub
    systems might call the publisher the "source". Typically a website or a web API, but can also be
    a web browser.

 *  Subscriber: A client application that subscribes to real-time updates of topics. Typically a
    Progressive Web App or a Mobile App, but can also be a server.

 *  Target: A subscriber, or a group of subscribers. A publisher is able to securely dispatch
    updates to specific targets. The target **MUST** be identified by an IRI [@!RFC3987] or by a
    string. Using an HTTPS [@!RFC7230] or HTTP [@!RFC7230] URI is **RECOMMENDED**.

 *  Hub: A server that handles subscription requests and distributes the content to subscribers when
    the corresponding topics have been updated. Any hub **MAY** implement its own policies on who
    can use it.

# Discovery

The discovery mechanism aims at identifying at least 2 URLs.

1.  The URL of one or more hubs designated by the publisher.

2.  The canonical URL for the topic to which subscribers are expected to use for subscriptions.

The URL of the hub **SHOULD** should be the "well-known" [@!RFC5785] fixed path
`/.well-known/mercure`.

If the publisher is a server, it **SHOULD** advertise the URL of one or more hubs to the subscriber,
allowing it to receive live updates when topics are updated. If more than one hub URL is specified,
the publisher **MUST** notifies each hub, so the subscriber **MAY** subscribe to one or more of
them.

Note: Publishers may wish to advertise and publish to more than one hub for fault tolerance and
redundancy. If one hub fails to propagate an update to the document, then using multiple independent
hub is a way to increase the likelihood of delivery to subscribers. As such, subscribers may
subscribe to one or more of the advertised hubs.

The publisher **SHOULD** include at least one Link Header [@!RFC5988] with `rel=mercure` (a hub link
header). The target URL of these links **MUST** be a hub implementing the Mercure protocol.

The publisher **MAY** provide the following target attributes in the Link Headers:

 *  `last-event-id`: the globally unique identifier of the last event dispatched by the publisher
    at the time of the generation of this resource. If provided, it **MUST** be passed to the
    hub through a query parameter called `Last-Event-ID` and will be used to ensure that possible
    updates having been made during between the resource generation time and the connection to the
    hub are not lost. See (#reconciliation). If this attribute is provided, the publisher **MUST**
    always set the `id` parameter when sending updates to the hub.

 *  `content-type`: the content type of the updates that will pushed by the hub. If omitted, the
    subscriber **MUST** assume that the content type will be the same as that of the original
    resource. Setting the `content-type` attribute is especially useful to hint that partial updates
    will be pushed, using formats such as JSON Patch [@RFC6902] or JSON Merge Patch [@RFC7386].

 *  `key-set=<JWKS>`: the key(s) to decrypt updates encoded in the JWKS (JSON Web Key Set) format.
    See (#encryption).

All these attributes are optional.

The publisher **MAY** also include one Link Header [@!RFC5988] with `rel=self` (the self link
header). It **SHOULD** contain the canonical URL for the topic to which subscribers are expected
to use for subscriptions. If the Link with `rel=self` is omitted, the current URL of the resource
**MUST** be used as a fallback.

Minimal example:

~~~ http
GET /books/foo HTTP/1.1
Host: example.com

HTTP/1.1 200 OK
Content-type: application/ld+json
Link: <https://example.com/.well-known/mercure>; rel="mercure"

{"@id": "/books/foo", "foo": "bar"}
~~~

Links embedded in HTML or XML documents as defined in the WebSub recommendation
[@W3C.REC-websub-20180123] **MAY** also be supported by subscribers. If both a header and an
embedded link are provided, the header **MUST** be preferred.

## Content Negotiation

For practical purposes, it is important that the `rel=self` URL only offers a single representation.
As the hub has no way of knowing what Media Type ([@RFC6838]) or language may have been requested
by the subscriber upon discovery, it would not be able to deliver the content using the appropriate
representation of the document.

It is, however, possible to perform content negotiation by returning an appropriate `rel=self`
URL according to the HTTP headers used in the initial discovery request. For example, a request
to `/books/foo` with an `Accept` header containing `application/ld+json` could return a `rel=self`
value of `/books/foo.jsonld`.

The example below illustrates how a topic URL can return different `Link` headers depending on the
`Accept` header that was sent.

~~~ http
GET /books/foo HTTP/1.1
Host: example.com
Accept: application/ld+json

HTTP/1.1 200 OK
Content-type: application/ld+json
Link: </books/foo.jsonld>; rel="self"
Link: <https://example.com/.well-known/mercure>; rel="mercure"

{"@id": "/books/foo", "foo": "bar"}
~~~

~~~ http
GET /books/foo HTTP/1.1
Host: example.com
Accept: text/html

HTTP/1.1 200 OK
Content-type: text/html
Link: </books/foo.html>; rel="self"
Link: <https://example.com/.well-known/mercure>; rel="mercure"

<!doctype html>
<title>foo: bar</title>
~~~

Similarly, the technique can also be used to return a different `rel=self` URL depending on the
language requested by the `Accept-Language` header.

~~~ http
GET /books/foo HTTP/1.1
Host: example.com
Accept-Language: fr-FR

HTTP/1.1 200 OK
Content-type: application/ld+json
Content-Language: fr-FR
Link: </books/foo-fr-FR.jsonld>; rel="self"
Link: <https://example.com/.well-known/mercure>; rel="mercure"

{"@id": "/books/foo", "foo": "bar", "@context": {"@language": "fr-FR"}}
~~~

# Subscription

The subscriber subscribes to a URL exposed by a hub to receive updates from one or many topics.
To subscribe to updates, the client opens an HTTPS connection following the Server-Sent Events
specification [@!W3C.REC-eventsource-20150203] to the hub's subscription URL advertised by the
publisher. The `GET` HTTP method must be used. The connection **SHOULD** use HTTP/2 to leverage
mutliplexing and other advanced features of this protocol.

The subscriber specifies the list of topics to get updates from by using one or several query
parameters named `topic`. The value of these query parameters **SHOULD** be a URI templates
[@!RFC6570].

Note: a URL is also a valid URI template.

The protocol doesn't specify the maximum number of `topic` parameters that can be sent, but the hub
**MAY** apply an arbitrary limit.

[The EventSource JavaScript
interface](https://html.spec.whatwg.org/multipage/server-sent-events.html#the-eventsource-interface)
**MAY** be used to establish the connection. Any other appropriate mechanism
including, but not limited to, readable streams [@W3C.NOTE-streams-api-20161129] and
[XMLHttpRequest](https://xhr.spec.whatwg.org/) (used by popular polyfills) **MAY** also be used.

The hub sends to the subscriber updates for resources matching the provided URI templates and the
provided targets (see (#authorization)). If the value of a `topic` parameter is not a valid URI
template, then the provided identifier **MUST** match exactly the resource identifier. If no targets
are specified, the update is dispatched to all subscribers. The hub **MUST** send these updates as
`text/event-stream` compliant events [!@W3C.REC-eventsource-20150203].

The `data` property **MUST** contain the new version of the topic. It can be the full resource, or a
partial update by using formats such as JSON Patch [@RFC6902] or JSON Merge Patch [@RFC7386].

All other properties defined in the Server-Sent Events specification **MAY** be used and **SHOULD**
be supported by hubs.

The resource **SHOULD** be represented in a format with hypermedia capabilities such as
JSON-LD [@W3C.REC-json-ld-20140116], Atom [@RFC4287], XML [@W3C.REC-xml-20081126] or HTML
[@W3C.REC-html52-20171214].

Web Linking [@!RFC5988] **SHOULD** be used to indicate the IRI of the resource sent in the event.
When using Atom, XML or HTML as the serialization format for the resource, the document **SHOULD**
contain a `link` element with a `self` relation containing the IRI of the resource. When using
JSON-LD, the document **SHOULD** contain an `@id` property containing the IRI of the resource.

Example:

~~~ javascript
// The subscriber subscribes to updates
// for the https://example.com/foo topic
// and to any topic matching https://example.com/books/{name}
const url = new URL('https://example.com/.well-known/mercure');
url.searchParams.append('topic', 'https://example.com/foo');
url.searchParams.append('topic', 'https://example.com/bar/{id}');

const eventSource = new EventSource(url);

// The callback will be called every time an update is published
eventSource.onmessage = function ({data}) {
    console.log(data);
};
~~~

The hub **MAY** require that subscribers are authorized to receive updates.

# Publication

The publisher sends updates by issuing `POST` HTTPS requests on the hub URL. When it receives an
update, the hub dispatches it to subscribers using the established server-sent events connections.

An application **CAN** send events directly to subscribers without using an external hub server, if
it is able to do so. In this case, it **MAY NOT** implement the endpoint to publish updates.

The request **MUST** be encoded using the `application/x-www-form-urlencoded` format and contains
the following data:

 *  `topic`: IRIs of the updated topic. If this key is present several times, the first occurrence
    is considered to be the canonical URL of the topic, and other ones are considered to be
    alternate URLs. The hub **MUST** dispatch this update to subscribers that are subscribed to both
    canonical or alternate URLs.

 *  `data`: the content of the new version of this topic.

 *  `target` (optional): target audience of this update. This key can be present several times. See
    (#authorization) for further information.

 *  `id` (optional): the topic's revision identifier: it will be used as the SSE's `id` property. If
    omitted, the hub **MUST** generate a valid globally unique id. It **MAY** be a UUID [@RFC4122].
    Even if provided, the hub **MAY** ignore the id provided by the client and generate its own id.

 *  `type` (optional): the SSE's `event` property (a specific event type).

 *  `retry` (optional): the SSE's `retry` property (the reconnection time).

In the event of success, the HTTP response's body **MUST** be the `id` associated to this update
generated by the hub and a success HTTP status code **MUST** be returned. The publisher **MUST** be
authorized to publish updates. See (#authorization).

# Authorization

To ensure that they are authorized, both publishers and subscribers must present a valid JWS
[@!RFC7515] in compact serialization to the hub. This JWS **SHOULD** be short-lived, especially
if the subscriber is a web browser. A different key **MAY** be used to sign subscribers' and
publishers' tokens.

Two mechanisms are defined to present the JWS to the hub:

 *  using an `Authorization` HTTP header

 *  using a cookie

If the publisher or the subscriber is not a web browser, it **SHOULD** use an `Authorization`
HTTP header. This `Authorization` header **MUST** contain the string `Bearer` followed by the JWS.
The hub will check that the JWS conforms to the rules (defined later) ensuring that the client is
authorized to publish or subscribe to updates.

By the `EventSource` specification [@W3C.REC-eventsource-20150203], web browsers
can not set custom HTTP headers for such connections, and they can only be
established using the `GET` HTTP method. However, cookies are supported and
can be included even in cross-domain requests if [the CORS credentials are
set](https://html.spec.whatwg.org/multipage/server-sent-events.html#dom-eventsourceinit-withcredentials):

If the publisher or the subscriber is a web browser, it **SHOULD** send a cookie called
`mercureAuthorization` containing the JWS when connecting to the hub.

Whenever possible, the `mercureAuthorization` cookie **SHOULD** be set during the discovery to
improve the overall security. Consequently, if the cookie is set during the discovery, both the
publisher and the hub have to share the same second level domain. The `Domain` attribute **MAY** be
used to allow the publisher and the hub to use different subdomains. See (#discovery).

The cookie **SHOULD** have the `Secure`, `HttpOnly` and `SameSite` attributes set. The cookie's
`Path` attribute **SHOULD** also be set to the hub's URL. See (#security-considerations).

When using authorization mechanisms, the connection **MUST** use an encryption layer such as HTTPS.

If both an `Authorization` HTTP header and a cookie named `mercureAuthorization` are presented by
the client, the cookie **MUST** be ignored. If the client tries to execute an operation it is not
allowed to, a 403 HTTP status code **SHOULD** be returned.

## Publishers

Publishers **MUST** be authorized to dispatch updates to the hub, and **MUST** prove that they are
allowed to send updates.

To be allowed to publish an update, the JWT presented by the publisher **MUST** contain a claim
called `mercure`, and this claim **MUST** contain a `publish` key. `mercure.publish` **MUST**
contain an array of targets the publisher is allowed to dispatch updates to.

If `mercure.publish`:

 *  is not defined, then the publisher **MUST NOT** be authorized to dispatch any update

 *  contains an empty array, then the publisher is only allowed to dispatch public updates

 *  contains the reserved string `*` as an array value, then the publisher is authorized to dispatch
    updates to all targets

If a topic is not public, the `POST` request sent by the publisher to the hub **MUST** contain a
list of keys named `target`. Their values **MUST** be of type `string`, and it is **RECOMMENDED** to
use valid IRIs. They can be, for instance, a user ID or a list of group IDs. If an update contains
at least one target the publisher is not authorized for, the hub **MUST NOT** dispatch the update
(even if some targets in the list are allowed) and **SHOULD** return a 403 HTTP status code.

## Subscribers

Subscribers **MAY** need to be authorized to connect to the hub. To receive updates destined to
specific targets, they **MUST** be authorized, and **MUST** prove they belong to at least one of the
specified targets. If the subscriber is not authorized, it **MUST NOT** receive any update having at
least one target.

To receive updates destined for specific targets, the JWS presented by the subscriber **MUST** have
a claim named `mercure` with a key named `subscribe` that contains an array of strings: a list of
targets the user is authorized to receive updates for. The targets **SHOULD** be IRIs.

If at least one target is specified, the update **MUST NOT** be sent to the subscriber by the hub,
unless the `mercure.subscribe` array of the JWS presented by the subscriber contains at least one of
the specified targets.

If the `mercure.subscribe` array contains the reserved string value `*`, then the subscriber is
authorized to receive updates destined for all targets.

# Reconnection and State Reconciliation {#reconciliation}

To allow re-establishment in case of connection lost, events dispatched by the hub **SHOULD**
include an `id` property. The value contained in this `id` property **SHOULD** be a globally unique
identifier. To do so, a UUID [@RFC4122] **MAY** be used.

According to the server-sent events specification, in case of connection
lost the subscriber will try to automatically re-connect. During the
re-connection, the subscriber **MUST** send the last received event id in a
[Last-Event-ID](https://html.spec.whatwg.org/multipage/iana.html#last-event-id) HTTP header.

The server-sent events specification doesn't allow this HTTP header to be set during the first
connection (before a reconnection). In order to fetch any update dispatched between the initial
resource generation by the publisher and the connection to the hub, the subscriber **MUST** send the
event id provided during the discovery in the `last-event-id` link's attribute in a query parameter
named `Last-Event-ID` when connecting to the hub. See (#discovery).

If both the `Last-Event-ID` HTTP header and the query parameter are present, the HTTP header
**MUST** take precedence.

If the `Last-Event-ID` HTTP header or query parameter exists, the hub **SHOULD** send all events
published following the one bearing this identifier to the subscriber.

The hub **MAY** discard some messages for operational reasons. The subscriber **MUST NOT** assume
that no update will be lost, and **MUST** re-fetch the original topic to ensure this (for instance,
after a long disconnection time).

The hub **MAY** also specify the reconnection time using the `retry` key, as specified in the
server-sent events format.

# Subscription Events

The hub **MAY** publish an update when a subscription to a topic is created or terminated. If this
feature is implemented by the hub, an update **MUST** be dispatched every time that a subscription
is created or terminated, and for each topic to which the client subscribes.

The topic of this update **MUST** follow the pattern
`https://mercure.rocks/subscriptions/{topic}/{subscriptionID}` where `topic` is the URL-encoded
value of the subscribed topic and `subscriptionID` is an unique identifier for this subscription.
`subscriptionID` **MAY** be a UUID [@RFC4122].

The content of the update **MUST** be a JSON-LD [@!W3C.REC-json-ld-20140116] document containing at
least the following properties:

 *  `@id`: the identifier of this update, it **MUST** be the same value as the subscription update's
    topic

 *  `@type`: the fixed value `https://mercure.rocks/Subscription`

 *  `topic`: the topic to which the subscription refers

 *  `active`: `true` when the subscription is created, and `false` when it is terminated

 *  `subscribe`: the subscription targets provided by the subscriber, see (#authorization)

 *  `publish`: the publication targets provided by the subscriber, see (#authorization)

 *  `address` (optional): the IP address ([@!RFC0791], [@!RFC8200]) of the subscriber

The JSON-LD document **MAY** contain other properties.

In order to only allow authorized subscribers to receive subscription events, the subscription
update **MUST** be marked as intended for subscribers providing the following targets:

 *  the fixed value `https://mercure.rocks/targets/subscriptions`

 *  a URL following the pattern `https://mercure.rocks/targets/subscriptions/{topic}` where topic is
    the URL-encoded value of the subscribed topic

# Encryption

Using HTTPS does not prevent the hub from accessing the update's content. Depending of the intended
privacy of information contained in the update, it **MAY** be necessary to prevent eavesdropping by
the hub.

To make sure that the message content can not be read by the hub, the publisher **MAY** encode the
message before sending it to the hub. The publisher **SHOULD** use JSON Web Encryption [@!RFC7516]
to encrypt the update content. The publisher **MAY** provide the relevant encryption key(s) in the
`key-set` attribute of the Link HTTP header during the discovery. See (#discovery). The `key-set`
attribute **SHOULD** contain a key encoded using the JSON Web Key Set [@!RFC7517] format. Any
other out-of-band mechanism **MAY** be used instead to share the key between the publisher and the
subscriber.

Update encryption is considered a best practice to prevent mass surveillance. This is especially
relevant if the hub is managed by an external provider.

# IANA Considerations

## Well-Known URIs Registry

A new "well-known" URI as described in (#discovery) has been registered in the "Well-Known URIs"
registry as described below:

 *  URI Suffix: mercure

 *  Change Controller: IETF

 *  Specification document(s): This specification, (#discovery)

 *  Related information: N/A

## Link Relation Types Registry

A new "Link Relation Type" as described in (#discovery) has been registered in the "Link Relation
Type" registry with the following entry:

 *  Relation Name: mercure

 *  Description: The Mercure Hub to use to subscribe to updates of this resource.

 *  Reference: This specification, (#discovery)

# Security Considerations

The confidentiality of the secret key(s) used to generate the JWTs is a primary concern. The
secret key(s) **MUST** be stored securely. They **MUST** be revoked immediately in the event of
compromission.

Possessing valid JWTs allows any client to subscribe, or to publish to the hub. Their
confidentiality **MUST** therefore be ensured. To do so, JWTs **MUST** only be transmitted over
secure connections.

Also, when the client is a web browser, the JWT **SHOULD** not be made accessible
to JavaScript scripts for resilience against [Cross-site Scripting (XSS)
attacks](https://owasp.org/www-community/attacks/xss/). It's the main reason why, when the client
is a web browser, using `HttpOnly` cookies as the authorization mechanism **SHOULD** always be
preferred.

In the event of compromission, revoking JWTs before their expiration is often difficult. To that
end, using short-lived tokens is strongly **RECOMMENDED**.

The publish endpoint of the hub may be targeted by [Cross-Site Request Forgery (CSRF)
attacks](https://owasp.org/www-community/attacks/csrf) when the cookie-based authorization mechanism
is used. Therefore, implementations supporting this mechanism **MUST** mitigate such attacks.

The first prevention method to implement is to set the `mercureAuthorization`
cookie's `SameSite` attribute. However, [some web browsers still not support this
attribute](https://caniuse.com/#feat=same-site-cookie-attribute) and will remain vulnerable.
Additionally, hub implementations **SHOULD** use the `Origin` and `Referer` HTTP headers set by web
browsers to verify that the source origin matches the target origin. If none of these headers are
available, the hub **SHOULD** discard the request.

CSRF prevention techniques, including those previously mentioned, are described
in depth in [OWASP's Cross-Site Request Forgery (CSRF) Prevention Cheat
Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html).

{backmatter}

# Acknowledgements

Parts of this specification, especially (#discovery) have been adapted from the WebSub
recommendation [@W3C.REC-websub-20180123]. The editor wish to thanks all the authors of this
specification.
