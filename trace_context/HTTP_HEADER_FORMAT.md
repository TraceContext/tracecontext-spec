# Trace context HTTP headers format

This section describes the binding of the distributed trace context to `traceparent`
and `tracestate` http headers.

## Relationship between the headers

The `traceparent` header represents the incoming request in a tracing system in
a common format. The `tracestate` header includes the parent in a potentially
vendor-specific format.

For example, a client traced in the congo system adds the following headers
to an outbound http request.
```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
tracestate: congo=BleGNlZWRzIHRohbCBwbGVhc3VyZS4=
```

If the receiving server is traced in the `rojo` tracing system, it carries
the over the state it received and adds a new entry with the position in
its trace.
```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: rojo=00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01,congo=lZWRzIHRoNhcm5hbCBwbGVhc3VyZS4=
```

You'll notice that the `rojo` system reuses the value of `traceparent` in its
entry in `tracestate`. This means it is a generic tracing system. Otherwise,
`tracestate` entries are opaque.

If the receiving server of the above is `congo` again, it continues from its
last position, overwriting its entry with one representing the new parent.

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b9c7c989f97918e1-01
tracestate: congo=Rpbmd1aXNoZWQsIG5vdCBvbmx5IGJ5IGhpcyByZWF=,rojo=00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

Notice when `congo` wrote its `traceparent` entry, it reuses the last trace ID
which helps in consistency for those doing correlation. However, the value of
its entry `tracestate` is opaque and different. This is ok.

Finally, you'll see `tracestate` retains an entry for `rojo` exactly as it was,
except pushed to the right. The left-most position lets the next server know
which tracing system corresponds with `traceparent`. In this case, since
`congo` wrote `traceparent`, its `tracestate` entry should be left-most.

# TraceParent field

## Header name

In order to increase interoperability across multiple protocols and encourage successful integration by default it is recommended to keep the header name lower case. Header name is a single word without any delimiters like hyphen (`-`).

Header name: `traceparent`

Platforms and libraries MUST expect header name in any casing and SHOULD send header name in lower case.

## Field value

This section uses the Augmented Backus-Naur Form (ABNF) notation of [RFC5234](https://tools.ietf.org/html/rfc5234), including the HEXDIG rules from that document.

```
value           = version "-" version-format
version         = 2HEXDIG   ; this document assumes version 00. Version 255 is forbidden
```

The value is US-ASCII encoded (which is UTF-8 compliant). Character `-` is
used as a delimiter between fields.

Version (`version`) is a 1 byte representing an 8-bit unsigned integer. Version 255 is invalid. Current specification assumes the `version` is set to `00`.

The following `version-format` definition used for version `00`.

```
version-format   = trace-id "-" span-id "-" trace-options

trace-id         = 32HEXDIG  ; 16 bytes array identifier. All zeroes forbidden
span-id          = 16HEXDIG  ; 8 bytes array identifier. All zeroes forbidden
trace-options    = 2HEXDIG   ; 8 bit flags. Currently only one bit is used. See below for details
```

### Trace-id

Is the ID of the whole trace forest. It is represented as a 16-bytes array, for example, 
`4bf92f3577b34da6a3ce929d0e0e4736`. All bytes `0` is considered invalid.

Implementation MAY decide to completely ignore the traceparent when the trace-id is invalid.

### Span-id

Is the ID of the caller span (parent). It is represented as an 8-bytes array, for example, 
`00f067aa0ba902b7`. All bytes `0` is considered invalid.

Implementation may decide to completely ignore the traceparent when the span-id is invalid.

## Trace-options

An [8-bit field](https://en.wikipedia.org/wiki/Bit_field) that controls tracing options such
as sampling, trace level etc. These flags are recommendations given by the caller rather than
strict rules to follow for three reasons:

1. Trust and abuse.
2. Bug in caller
3. Different load between caller service and callee service might force callee to down sample.

Like other fields, `trace-options` is hex-encoded. For example, all 8 flags set would be 'ff'
and no flags set would be '00'.

As this is a bit field, you cannot interpret flags by decoding the hex value and looking at
the resulting number. For example, a flag `00000001` could be encoded as `01` in hex, or `09`
in hex if present with the flag `00001000`. A common mistake in bit fields is forgetting to
mask when interpreting flags.

Here is an example of properly handing trace options:
```java
static final byte FLAG_TRACED = 1; // 00000001
...
boolean traced = (traceOptions & FLAG_TRACED) == FLAG_TRACED
```

#### Traced Flag (00000001)
When set, the least significant bit recommends the request should be traced. A caller who
defers a tracing decision leaves this flag unset.

#### Other Flags
The behavior of other flags, such as (00000010) are undefined.

## Examples of HTTP headers

*Valid sampled traceparent:*

```
Value = 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
base16(<Version>) = 00
base16(<TraceId>) = 4bf92f3577b34da6a3ce929d0e0e4736
base16(<SpanId>) = 00f067aa0ba902b7
base16(<TraceOptions>) = 01  // sampled
```

*Valid not-sampled traceparent:*

```
Value = 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00
base16(<Version>) = 00
base16(<TraceId>) = 4bf92f3577b34da6a3ce929d0e0e4736
base16(<SpanId>) = 00f067aa0ba902b7
base16(<TraceOptions>) = 00  // not-sampled
```

# TraceState field

The `tracestate` HTTP header field conveys information about request position in multiple distributed tracing graphs.

## Header name

In order to increase interoperability across multiple protocols and encourage successful integration by default it is recommended to keep the header name lower case. Header name is a single word without any delimiters like hyphen (`-`).

Header name: `tracestate`

Platforms and libraries MUST expect header name in any casing and SHOULD send header name in lower case.

## Header value

This section uses the Augmented Backus-Naur Form (ABNF) notation of [RFC5234](https://tools.ietf.org/html/rfc5234), including the DIGIT rule in [appendix B.1 for RFC5234](https://tools.ietf.org/html/rfc5234#appendix-B.1). It also includes the OWS rule from [RFC7230 section 3.2.3](https://tools.ietf.org/html/rfc7230#section-3.2.3).

`DIGIT` rule defines number `0`-`9`.

The `OWS` rule defines an optional whitespace. It is used where zero or more whitespace characters might appear. When it is preferred to improve readability - a sender SHOULD generate the optional whitespace as a single space; otherwise, a sender SHOULD NOT generate optional whitespace. See details in corresponding RFC.

The `tracestate` field value is a `list` as defined below. The `list` is a series of `list-members` separated by commas `,`, and a `list-member` is a key/value pair separated by an equals sign `=`. Spaces and horizontal tabs surrounding `list-member`s are ignored. There can be a maximum of 32 `list-member`s in a `list`.

A simple example of a `list` with two `list-member`s might look like: `vendorname1=opaqueValue1,vendorname2=opaqueValue2`.


```
list  = list-member 0*31( OWS "," OWS list-member )
list-member = key "=" value
```

Identifiers are short (up to 256 characters) textual identifiers.

```
key = lcalpha 0*255( lcalpha / DIGIT / "_" / "-"/ "*" / "/" )
lcalpha    = %x61-7A ; a-z
```

Note that identifiers MUST begin with a lowercase letter, and can only contain lowercase letters `a`-`z`, digits `0`-`9`, underscores `_`, dashes `-`, asterisks `*`, and forward slashes `/`.

Value is opaque string up to 256 characters printable ASCII [RFC0020](https://www.rfc-editor.org/info/rfc20) characters (i.e., the range 0x20 to 0x7E) except comma `,` and `=`. Note that this also excludes tabs, newlines, carriage returns, etc.

```
value    = nblk-chr 0*254(chr) nblk-chr
nblk-chr = %x21-2B / %x2D-3C / %x3E-7E
chr      = %x20 / nblk-chr
```

The length of a combined header MUST be less than or equal to 512 bytes. If the length of a combined header is more than 512 bytes it SHOULD be ignored.

Example: `vendorname1=opaqueValue1,vendorname2=opaqueValue2`

The value a concatenation of trace graph key-value pairs. Only one entry per
key is allowed because the entry represents that last position in the trace.
Hence implementors must overwrite their entry upon reentry to their tracing
system.

For example, if tracing system name is `congo`, and a trace started in their
system, went through a system named `rojo` and later returned to `congo`, the
`tracestate` value would not be:

`congo=congosFirstPosition,rojo=rojosFirstPosition,congo=congosSecondPosition`

Rather, the entry would be rewritten to only include the most recent position:
`congo=congosSecondPosition,rojo=rojosFirstPosition`

**Limits:**
There might be multiple `tracestate` headers in a single request according to [RFC7230 section 3.2.2](https://tools.ietf.org/html/rfc7230#section-3.2.2). Maximum length of a combined header MUST be less than 512 characters. This length includes commas required to separate list items. But SHOULD NOT include optional white space (OWA) characters.

`tracestate` field contains essential information for requests correlation. Platforms and tracing systems MUST propagate this header. Compliance with specification will require storing of `tracestate` as part of request payload or associated metadata. Allowing the long field values can make compliance to the specification impossible. Thus, the aggressive limit of 512 characters was chosen.

If the `tracestate` value has more than 512 characters, the tracer CAN decide to forward the `tracestate`. When propagating `tracestate` with the excessive length - the assumption SHOULD be that the receiver will drop this header.

## Name format

Name starts with the beginning of the string or separator `,` and ends with the
equal sign `=`. The contents of the name are any URL encoded string that does
not contain an equal sign `=`. Names should intuitively identify the tracing
system even if multiple systems per vendor are present.

## Value format

Value starts after equal sign and ends with a separator `,` or end of string.
In the case of a generic tracing system, it contains the same data as the most
recent `traceparent` value. Other systems may have different formatting, such
as Base64 encoded opaque values.

# Examples of HTTP headers

Single tracing system (generic format): 

```
tracestate: rojo=00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

Multiple tracing systems (with different formatting):

```
tracestate: rojo=00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01,congo=lZWRzIHRoNhcm5hbCBwbGVhc3VyZS4=
```

# Mutating the traceparent field

## Base mutations

Library or platform receiving `traceparent` request header MUST send it to outgoing requests. It MAY mutate the value of this header before passing to outgoing requests.

If the value of `traceparent` field wasn't changed before propagation - `tracestate` MUST NOT be modified as well. Unmodified headers propagation is typically implemented in pass-thru services like proxies. This behavior may also be implemented in a service which currently does not collect distributed tracing information.

Here is the list of allowed mutations:

1. **Update `span-id`**. The value of property `span-id` can be regenerated. This is the most typical mutation and should be considered a default.
2. **Mark trace for sampling**. The value of `sampled` flag of `trace-options` may be set to `1` if it had value `0` before. `span-id` MUST be regenerated with the `sampled` flag update. This mutation typically happens to mark the importance of a current distributed trace collection.
3. **Restarting trace**. All properties - `trace-id`, `span-id`, `trace-options` are regenerated. This mutation is used in the services defined as a front gate into secure network and eliminates a potential denial of service attack surface. 

Libraries and platforms MUST NOT make any other mutations to the `traceparent` header.

# Mutating the tracestate field

Library or platform receiving `tracestate` request header MUST send it to outgoing requests. It MAY mutate the value of this header before passing to outgoing requests. The main concept of `tracestate` mutations is that order of unmodified key-value pairs MUST be preserved. Modified keys MUST be moved to the beginning of the list.

Here is the list of allowed mutations:

1. **Update key value**. The value of any key can be updated. Modified key MUST be moved to the beginning of the list. This is the most common mutation resuming the trace.
2. **Add new key-value pair**. New key-value pair should be added into the beginning of the list.
3. **Delete the key-value pair**. Any key-value pair MAY be deleted. It is highly discouraged to delete keys that wasn't generated by the same tracing system or platform. Deletion of unknown key-value pair will break correlation in other system. This mutation enables two scenarios. First is proxies to block certain `tracestate` keys for privacy and security concerns. Second scenario is a truncation of long `tracestate`.

