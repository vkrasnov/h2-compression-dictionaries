---
title: Compression Dictionaries for HTTP/2
docname: draft-vkrasnov-h2-compression-dictionaries
date: 2017-03
category: info

ipr:
area: General
workgroup:
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -  ins: V. Krasnov
    name: Vlad Krasnov
    organization: Cloudflare Inc.
    email: vlad@cloudflare.com

normative:
  RFC2119:
  RFC2616:
  RFC7540:
informative:
  RFC1951:
  RFC7932:
  BREACH:
    title: "BREACH: SSL, Gone in 30 Seconds"
    author:
      -
        ins: A. Prado
        name: Angelo Prado
      -
        ins: N. Harris
        name: Neal Harris
      -
        ins: Y. Gluck
        name: Yoel Gluck
    date: 2013
    target: http://breachattack.com/



--- abstract

This document specifies a new HTTP/2 frame type and new HTTP/2 settings values
that would enable the use of previously transferred data as compression
dictionaries, significantly improving overall compression ratio for a given
connection.

In addition, this document proposes to define a set of industry standard, static,
dictionaries to be used with any Lempel-Ziv based compression for the common
textual MIME types prevalent on the web.


--- middle

# Introduction

The HTTP/2 {{RFC7540}} protocol encourages the use of many small assets for
CSS/JS/HTML, due to its multiplexed nature. Prior to HTTP/2, asset inlining was
encouraged, resulting in fewer, larger assets per website.

The HTTP/2 protocol also allows for transmitted data to be compressed with a
lossless compression format. The format used is specified in the
"Content-Encoding" (see {{RFC2616}}, section 14.11) header field. For example,
"Content-Encoding: br" means the data was compressed using the Brotli format.

The nature of the compression algorithms, such as DEFLATE {{RFC1951}} and
Brotli {{RFC7932}}, used with HTTP in practice, require a certain "window" of
data to perform backward matching. Therefore, larger files have much better
compression ratio. To improve compression for smaller files, these
algorithms allow to use a chunk of arbitrary data as a "Custom Dictionary"
and function as the initial sliding window.

While compression, especially of dynamic resources, is a compute-heavy operation,
where investing more compute power results in diminishing returns (in terms of
compression ratio). This technique is known to improve compression ratio
significantly, while not requiring significant additional processing time, and
is supported by most LZ based compression formats.

This document introduces a mechanism for using previously transmitted data over
HTTP/2 as a dictionary to be used with an underlying compression algorithm.

## Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in RFC 2119 {{RFC2119}}.

# HTTP/2 Extension

## Extension Settings

The extension introduces two new SETTINGS values.

SETTINGS_COMPRESSION_SETTINGS(0xTBA):
: For greater compression, and to prevent setting identifier depletion, the
32-bit value for this setting is defined as follows:

~~~~
+---------------+---------+-----------+-----------+
| SDVersion (8) | Fmt (8) | DSize (8) | NDict (8) |
+---------------+---------+-----------+-----------+
~~~~

NDict:
: Indicates the number of dictionaries the client is willing to maintain.
The default value is 0, the maximal value is 255.

DSize:
: Log2 of the maximal size of each dictionary. The default value is 0, the
maximal value is 255. For example value of 17 indicates each dictionary MUST
be smaller or equal to 2^17 (131,072 octets).

Fmt:
: Compression format to use. 1 indicates brotli, 2 indicates zlib. The default
value is 0, for which cross-stream compression is disabled. Values 3-255 are
reserved for additional formats.

SDVersion:
: If greater than 0, indicates the version of static dictionaries to use.
Maximal value is 255, the default value is 0, which indicates no static
dictionaries are used.


## Extension Frames

### The SET_COMPRESSION_CONTEXT

The SET_COMPRESSION_CONTEXT frame (type=0xTBA).

~~~~
+-------------+
| Context (8) |
+-------------+
~~~~

The SET_COMPRESSION_CONTEXT frame can be sent by the client on any stream in
the idle state (prior to the first HEADER frames on the stream).
It indicates that the stream MAY be compressed by dictionaries with the given
Context ID.

The SET_COMPRESSION_CONTEXT frame contains the following fields:

Context:
: an 8-bit Context ID that indicates the compression context for the stream.
A stream can only be compressed by dictionaries with the same Context. If a
stream SETs a dictionary, it will be assigned the Context indicated by this
frame. If the frame is ommited, then the context value is assumed to be 0.
The allowed Context values are 0 through 253.
A special Context ID of 255 indicates the stream SHALL NOT be compressed at all
and passed as is.
A special Context ID of 254 indicates the stream SHALL NOT be compressed with
any dictionary, and it MUST NOT be used as dictionary for other streams.

### The SET_DICTIONARY Frame

The SET_DICTIONARY frame (type=0xTBA).

~~~~
+-------------+-------------+
| Dict ID (8) |   Size (8)  |
+-------------+-------------+
~~~~

The SET_DICTIONARY frame can be sent from the server to the client, on any client
initiated stream in the open or half-closed (remote) states. The SET_DICTIONARY
frame MUST precede any DATA frames on that stream. The SET_DICTIONARY frame SHOULD
be followed by sufficient DATA frames to fill Size octets after decompression,
even if an RST frame was received for that stream. If not enough DATA was sent,
the Dictionary for the given ID is considered uninitialized.

More than one SET_DICTIONARY frames MAY be sent on a given stream.

The SET_DICTIONARY frame contains the following fields:

Dict ID:
: an 8-bit ID, indicates the dictionary. MUST be lower than the value agreed by
the SETTINGS_COMPRESSION_SETTINGS setting.

Size:
: Indicates how many octets (as a power of 2) of the given stream will be used.
If Size is greater than the length of the transmitted data, then all of the data
will be used.

The SET_DICTIONARY frame defines the following flag:

APPEND (0x1):
: Indicates that the data is to be appended to the existing dictionary with the
given ID, as opposed to replacing it with the new data.


### The USE_DICTIONARY Frame

The USE_DICTIONARY frame (type=0xTBA).

~~~~
+-------------+
| Dict ID (8) |
+-------------+
~~~~

The USE_DICTIONARY frame indicates that the current stream is to be decompressed
with the indicated dictionary. The USE_DICTIONARY frame MUST be sent before any
DATA frame on a given stream. SET_DICTIONARY and USE_DICTIONARY frames MAY be
sent on the same stream. Only one USE_DICTIONARY frame MAY be sent for a stream.

The USE_DICTIONARY frame contains the following fields:


Dict ID:
: an 8-bit ID that indicates which dictionary to use. The dictionary MUST be
previously defined by a SET_DICTIONARY frame, or by a static dictionary.

## Static Dictionaries

This document proposes to generate a set of up to 8 standard dictionaries to be
optionally bundled with supporting implementations. Each dictionary should be
32,768 or 65,536 octets long.

Each static dictionary will be identified by an integer ID in the range {0..7}.

If either endpoint supports the use of static dictionaries, it will indicate this
by setting the SDVersion value of SETTINGS_COMPRESSION_SETTINGS to greater than
0. The number will indicate the highest version of the dictionaries known.

The actual version used will be the lowest of the two values set by the endpoints.

If the client and the server agreed on the use of static dictionaries, then
both will initialize the first 8 dictionaries (IDs 0 through 7), with the contents
of the static dictionaries. The Context ID for those dictionaries will be 0.

If the value of the field NDict is lower than 8, then up to NDict dictionaries
will be initialized.

# Dictionary State

Both the server and the client MUST process the SET_DICTIONARY and USE_DICTIONARY
frames in the order they are sent/received, with the exception when both are sent
over the same stream. In that case USE_DICTIONARY is processed prior to the
SET_DICTIONARY frames.

Doing otherwise will result in an illegal state of the dictionaries. This is
similar to the way HEADER frames are processed in order to maintain legal HPACK
state on the server and the client.

Initially the dictionaries are uninitialized, unless static dictionary use is
agreed upon by both endpoints. In that case the dictionaries are initialized
as described in {{static-dictionaries}}.

When SET_DICTIONARY is used with the APPEND flag cleared, both the server and
the client SHALL use the first Size octets of the decompressed stream as the
dictionary for the given ID. The Context ID for the dictionary is inherited
from the stream.

When SET_DICTIONARY is used with the APPEND flag set, both the server and the
client SHALL take the existing dictionary with the given ID, append the first
Size octets of the decompressed stream to it from the right, and use the last
2^DSize octets as the dictionary for the given ID. If the Context ID of the
dictionary before the appendage was different from the stream ID this is
considered as an error of type COMPRESSION_ERROR.

## Server Behavior

The server MAY send a SET_DICTIONARY frame on any client initiated stream in the
open or half-closed (remote) states, prior to sending any DATA on that stream.

After the server sends a SET_DICTIONARY stream with a given ID, it MUST not use
the dictionary for compression, until it sent sufficient data for the dictionary
to become usable by the client. Sufficient data is computed as the size of the
data after decompression.

After sufficient data was sent, the server MAY use the associated dictionary on
any subsequent stream by sending the USE_DICTIONARY frame.

The server MUST only use dictionaries on streams with the same Context ID.

## Client Behavior

If the client wants to use cross-stream compression it must take into
consideration that the data transferred might not be compressible. This can
happen when an application level compression is used, encrypted data is
transferred or for specific binary file formats, such as video, images etc.

To maximize the benefits of cross-stream compression, the client SHOULD
disable application level compression. In addition when binary data is expected
on the stream, the clinet SHOULD hint to the server by sending a
SET_COMPRESSION_CONTEXT with the special value of 255.

When receiving a USE_DICTIONARY frame, the client MUST use the specified
dictionary to decompress the DATA.

A given stream MAY receive a SET_DICTIONARY and USE_DICTIONARY with the same ID.
In that case the stream is decompressed with the existing dictionary and the
dictionary is updated using the decompressed data.

If a USE_DICTIONARY frame arrives for an uninitialized dictionary, this is
considered as stream error of type COMPRESSION_ERROR.

If a USE_DICTIONARY frame arrives for a stream with Context ID different from
the dictionary Context ID, this is considered as a stream error of type
COMPRESSION_ERROR.

# Security Considerations

As with any compression scheme, using cross-stream compression is potentially
more sensitive to {{BREACH}} type of attacks.

Therefore, this extension SHALL be disabled by default by all server
implementations.

If a server acts as an intermediary, then it MUST NOT enable cross-stream
compression, unless the origin also enables cross-stream compression. If the
origin does not use the HTTP/2 protocol, the intermediary server MUST NOT use
cross-stream compression, unless notified by the origin explicitly by either a
receipt of a pre-agreed upon HTTP header, or out-of-band. If the origin does use
the HTTP/2 protocol, then the intemediary server MUST preserve the context ids
from the client to the origin.

A client MAY indicate to the server a request SHALL NOT be compressed using a
dictionary or used as a dictionary by sending a SET_COMPRESSION_CONTEXT with the
special value of 254.

The client MAY also indicate to the server other Context values for any stream,
when it is desirable to prevent cross-stream side channel leaks, such as when
connection coalescing is used.

The server SHOULD avoid the use of cross-stream compression on cross-site
requests.

# HTTP/1.1 Mappings

Cross-stream compression by definition is very efficient for HTTP/2, because
it allows for a very fine-grained dictionary defention and consumption on the
protocol level, and also because during the life span of an HTTP/2 connection
it will serve more assets on average than an HTTP/1.1 connection.

However there are cases when a behavior similar to cross-stream compression is
desirable for HTTP/1.1 as well. This is especially true for long lived
connections, for example from a CDN server to origin server.

Therefore the following mapping to the HTTP/1.1 protocol is suggested.

## The mapping

The client SHALL indicate support for "cross-asset" compression by sending the
CSCS header with the first request for the connection:

~~~~
CSCS = "CSCS" ":"
1("nd" "=" value ";"
"ds" "=" value ";"
"fmt" "=" value ";"
"sd" "=" value)
~~~~

Where "nd", "ds", "fmt" and "sd" map to the NDict, DSize, Fmt, and SDVersion
settings.

It MAY then indicate the desired Context ID for the request with the CSCC header:

~~~~
CSCC = "CSCC" ":" 1(value)
~~~~

Where the values are the same as defined for the SET_COMPRESSION_CONTEXT frame.

The server MAY then compress the response body, similarly indicating the USE
and SET functions using the CSCD header:

~~~~
CSCD = "CSCD" ":"
["u" "=" value ";"]
0#("s" "=" value [ ";" "a" "=" value] )
~~~~

Where "u" indicates the dictionary id to use if any, maps to the USE_DICTIONARY
frame. "s" indicates one or more dictionary to set, maps to the SET_DICTIONARY
frame. The optional "a" value indicates if the dictionary is set in the "append"
mode, maps to the APPEND flag of SET_DICTIONARY frame.

Such mapping only exists for a single persistent HTTP/1.1 connection, and not
between multiple connections. It therfore requires only connection-level state
keeping.

--- back
