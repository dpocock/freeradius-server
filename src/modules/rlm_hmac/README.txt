

rlm_hmac works a bit like rlm_digest

It can use either a Cleartext-Password or a Digest-HA1 value.  It
looks for both of them.  It will use the Digest-HA1 if both
exist for the given user.

It is biased for STUN/TURN.  Specifically, this means that

- the HMAC key value is a H(A1) value (where H is MD5)

- the HMAC variant is HMAC-SHA1

It should be fairly trivial to generalise this module to
support other HMAC variants or key variants.  It is convenient to
store Digest-HA1 in a RADIUS server because it is used by SIP too.

Design considerations
---------------------

RADIUS only supports a limited number of attribute names.  On the wire,
the attribute is represented by a single byte.  Consequently, many protocols
use special schemes to embed extended attribute/value pairs in
a single attribute.

STUN/TURN servers don't just need to verify a HMAC from a client.  They
also need to sign the outgoing messages to clients.  For this purpose,
they need to receive the HMAC calculated by the RADIUS server,
a yes/no response is not sufficient.

TURN servers may require optional attributes in future, for example,
to specify user bandwidth limits.  These should be part of the design
even if they are not implemented now.

For SIP with RADIUS, RFC 5090 suggests that the RADIUS server should be
able to specify a nonce for a client (rather than the SIP server creating
and validating the nonce).  It may be desirable to implement such a
requirement for STUN/TURN with RADIUS.  The easiest implementation
(and the way SIP servers work with rlm_digest) simply involves the
NAS generating and validating nonces itself while the RADIUS server
only validates the hashes.

To do - essential
-----------------

Before the code can be part of any proper RADIUS server release,
the whole concept needs to be formalised and at the very minimum,
the IANA must reserve a code number for the HMAC-Code attribute.

http://www.ietf.org/assignments/radius-types/radius-types.xml

The maximum length of an attribute's value in RADIUS is 253
bytes.  It is theoretically possible for STUN/TURN packets to
exceed this length.  In this case, it is necessary to send the
message body to the RADIUS server by repeating the
HMAC-Body attribute multiple times and splitting the body across
the repeated attributes.  The issue is discussed here:

http://tools.ietf.org/html/draft-perez-radext-radius-fragmentation-06#section-1

To do - optional
----------------

Implementing other HMAC variants (e.g. HMAC-MD5)

Implementing other key variants (e.g. just using the cleartext password rather
than the H(A1) value)

Identify other network protocols that may benefit from HMAC in RADIUS

- Dynamic DNS updates can be authenticated with TSIG, HMAC can be used:
  http://tools.ietf.org/html/rfc4635
  This appears to be an interesting use case

- IPsec uses HMAC, but it is only used with transient keys, no need
  to integrated with RADIUS for HMAC

Testing
-------

It can be tested by adding something like this to 
the file raddb/mods-config/files/authorize:

test   Auth-Type := HMAC, Cleartext-Password := "foobar"
       Reply-Message = "hello world"

testha1	Auth-Type := HMAC, Digest-HA1 := "61616161616161616161616161616161"
	Reply-Message = "hello world"

and creating a file /tmp/hmac.req:

User-Name = "test",
HMAC-Realm = "example.org",
HMAC-User-Name = "test",
HMAC-Nonce = "01234567890123456789012345678901",
HMAC-Algorithm = "HMAC-SHA1",
HMAC-Body = "unimportant message"

radclient can then be invoked with something like this:

  radclient -f /tmp/hmac.req localhost auth testing123

The reply packet (Access-Accept) contains the HMAC-Code value.
It can be used to verify a HMAC from a peer or to sign a message
to be sent to the peer.  STUN/TURN needs to use the values in
both of these paradigms.

