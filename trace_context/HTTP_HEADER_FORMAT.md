# Trace Context HTTP Header Format

## Header name

`Trace-Context`

## Field value

```
base16(<version>)-<version_format>
```

The value is US-ASCII encoded (which is UTF-8 compliant). Character `-` is
used as a delimiter between fields.

### Version

Is a 1 byte representing an 8-bit unsigned integer. Version 255 reserved. Current specification assumes the `version` is set to `0`.

### Version 0 Format 

```
base16(<trace-id>)-base16(<span-id>)[-base16(<trace-options>)]
```

`trace-id` and `span-id` are required. The `trace-options` is optional. Character `-`
 is used as a delimiter between fields.

### Trace-id

Is the ID of the whole trace forest. It is represented as a 16-bytes array, for example, 
`4bf92f3577b34da6a3ce929d0e0e4736`. All bytes 0 is considered invalid.

Implementation MAY decide to completely ignore the trace-context when the trace-id is invalid.

### Span-id

Is the ID of the caller span (parent). It is represented as an 8-bytes array, for example, 
`00f067aa0ba902b7`. All bytes 0 is considered invalid.

Implementation may decide to completely ignore the trace-context when the span-id is invalid.

### Trace-options

Controls tracing options such as sampling, trace level etc. It is a 1 byte representing an 8-bit 
unsigned integer. The flags are recommendations given by the caller rather than strict rules to 
follow for three reasons:

1. Trust and abuse.
2. Bug in caller
3. Different load between caller service and callee service might force callee to down sample.    

### Bits behavior definition (01234567):
* The least significant bit (the 7th bit) provides recommendation whether the request should be 
traced or not (`1` recommends the request should be traced, `0` means the caller does not
make a decision to trace and the decision might be deferred). When `trace-options` is missing
the default value for this bit is `0`
* The behavior of other bits is currently undefined.

# Mutating the header

Library or platform receiving `Trace-Context` request header MUST send it to outgoing http requests. It MAY mutate the value of this header before passing to outgoing http requests. 

Library or platform receiving `Trace-Context` response header MAY send it as a response header to originating http request. Proxies, load balancers, and other platforms with 1:1 mapping of request and response SHOULD return this `Trace-Context` header to originating http request. `Trace-Context` response header MAY also be mutated.

Here is the list of allowed mutations:

1. **Update `span-id`**. The value of property `span-id` can be regenerated.
2. **Mark trace for sampling**. The value of `sampled` flag of `trace-options` may be set to `1` if it had value `0` before. `span-id` MUST be regenerated with the `sampled` flag update.
3. **Restarting trace**. All properties - `trace-id`, `span-id`, `trace-options` are regenerated. Library or platform MAY choose to return the newly generated `trace-id` as `Trace-Context` response header.

Libraries and platforms MUST NOT make any other mutations to the `Trace-Context` header.


# Examples of HTTP headers

*Valid sampled Trace-Context:*

```
Value = 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
base16(<Version>) = 00
base16(<TraceId>) = 4bf92f3577b34da6a3ce929d0e0e4736
base16(<SpanId>) = 00f067aa0ba902b7
base16(<TraceOptions>) = 01  // sampled
```

*Valid not-sampled Trace-Context:*

```
Value = 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00
base16(<Version>) = 00
base16(<TraceId>) = 4bf92f3577b34da6a3ce929d0e0e4736
base16(<SpanId>) = 00f067aa0ba902b7
base16(<TraceOptions>) = 00  // not-sampled
```
