# How to use DNS Wire Format for DNS over HTTPS queries. 

[DNS over HTTPs](https://datatracker.ietf.org/doc/html/rfc8484) (DoH) allows tamper-resistant client-side interrogation of DNS records. Some DNS providers, like Cloudflare, support [JSON-structured queries](https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-https/make-api-requests/dns-json/) which are quite simple to construct and interpret. However, most DNS providers supporting DoH require queries to be submitted in DNS Wire Format (DWF) as specified in [RFC 1035](https://datatracker.ietf.org/doc/html/rfc1035). 

While DWF is quite efficient in terms of communication overhead, it can be difficult to piece together if you are unfamiliar with it since there are few online resources that present it in an accessible fashion. This post is a  practical crash course in constructing DWF-compatible DNS queries to leverage DoH in you client-side applications. 

## Anatomy of a DNS Message

A user client can issue queries to any DNS provider that supports DNS over HTTPS via a vanilla fetch or curl call. However, queries must be structured as a [DNS message](https://datatracker.ietf.org/doc/html/rfc1035#section-4). A DNS message has 5 fields, each of which are binary encoded in groups of 2-byte frames:

- header - 6 frames (we'll construct this once and use it over and over)
- question field - 2 frames + variable byte length Query field (we'll walk through constructing this here)
- answer field - 5 frames + variable length Name & variable length Record Data (you'll only bother with this field in the DNS provider's response payload)
- authority field - 5 frames + variable length Name & variable length Record Data (you won't likely need this field)
- additional info field - 5 frames + variable length Name & variable length Record Data (you probably won't need this field either)

To leverage DoH in a user application, you need only bother with the header (since there must AlWAYS be a header) and question fields since you will only ever be sending a query message to a DNS provider rather than participating in the DNS system itself.

## The Header Field

The header contains 6 frames:

- 16 bits for an arbitrary reference id, we'll set this to 0 but you can put anything you want
- 16 bits for specifying various metadata
  - first bit specifies if the message is a query (0) or response (1)
  - the next four bits specify an opcode, we will always set them to 0 because we want to issue a query
  - bits 6 and 7 are only set for response messages
  - bit 8 tells the DoH server to pursue the resource record recursively, we'll always set this to 1
  - bit 9 is only set for response messages
  - bits 10 through 12 are reserved for a future use case of the RFC 1035 standard
  - bits 13 through 16 are a response code only set in response messages
- 16 bits for specifying the number of queries in the query field, we'll be setting this to 1
- 16 bits for specifying the number of resource records in the answer field, we'll be setting this to 0
- 16 bits for name resource record count, we'll be setting this to 0
- 16 bits for the number of additional info records in the additional info field

So since we will always be making a single query (not generating a response) and we want the DNS provider to recursively pursue the record, our header will likely always look like this (in hex encoded binary):

`00 00 01 00 00 01 00 00 00 00 00 00`

You can save this binary string and use it over and over for single DoH queries. 

## The Query Field

Next, the query field carries the information needed to pass along a complete question to a DNS provider. 

The first `n` frames encode the domain name, called the [`QNAME`](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.2), which the query is about. A domain name (like www.snickerdoodle.com) is written like this:

`3www13snickerdoodle3com0`

or in hex encoding like this:

`03 77 77 77 0d 73 6e 69 63 6b 65 72 64 6f 6f 64 6c 65 03 63 6f 6d 00`

The numbers tell the message receiver how to read the `QNAME` which is terminated with a byte of zeros. 

> [!NOTE] 
> The `QNAME` field does not need to be an integer number of frames, i.e, its length may be an odd number of bytes. There is no padding requried to fill out a full frame. 

The next frame after `QNAME`, is the type of query, called [`QTYPE`](https://datatracker.ietf.org/doc/html/rfc1035#section-3.2.2). If you want to fetch a TXT record, its `TYPE` is 16 (or `00 10` as a frame of hex). So, adding this to the end of our hex encoding from above for www.snickerdoodle.com, you'd have:

`03 77 77 77 0d 73 6e 69 63 6b 65 72 64 6f 6f 64 6c 65 03 63 6f 6d 00 00 10`

The final frame of your query section is the class of query, called [`QCLASS`](https://datatracker.ietf.org/doc/html/rfc1035#section-3.2.4). You will almost always be issuing an Internet class query, so its `CLASS` is 1 (or `00 01` as a frame of hex). Again concatenating this to the end of our hex encoded domain example gives us:

`03 77 77 77 0d 73 6e 69 63 6b 65 72 64 6f 6f 64 6c 65 03 63 6f 6d 00 00 10 00 01`

## Putting It All Together

Concatenating the header and query fields gives:

`00 00 01 00 00 01 00 00 00 00 00 00 03 77 77 77 0d 73 6e 69 63 6b 65 72 64 6f 6f 64 6c 65 03 63 6f 6d 00 00 10 00 01`

and [encoding to base 64](https://cryptii.com/pipes/base64-to-binary) gives us:

`AAABAAABAAAAAAAAA3d3dw1zbmlja2VyZG9vZGxlA2NvbQAAEAAB`

With this, you can construct an DWF-compatible DoH query for a TXT record attached to www.snickerdoodle.com that is supported by most major DNS providers.

### Cloudflare

```
echo -n 'AAABAAABAAAAAAAAA3d3dw1zbmlja2VyZG9vZGxlA2NvbQAAEAAB' | base64 --decode | curl -H 'content-type: application/dns-message' --data-binary @- https://cloudflare-dns.com/dns-query -o - | hexdump -C
```

```
await fetch("https://cloudflare-dns.com/dns-query?dns=AAABAAABAAAAAAAAA3d3dw1zbmlja2VyZG9vZGxlA2NvbQAAEAAB", {
  headers: {
    accept: "application/dns-message"
  }
})
```

### Google

```
await fetch("https://dns.google/dns-query?dns=AAABAAABAAAAAAAAA3d3dw1zbmlja2VyZG9vZGxlA2NvbQAAEAAB", {
  headers: {
    accept: "application/dns-message"
  }
})
```

> [!Important]
> You will run into [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) issues with some DoH servers. In that case your best course of action is probably to use a [CORS proxy](https://httptoolkit.com/blog/cors-proxies/) for your application. 

Now you have the knowledge necessary to construct DWF-compatible queries yourself so that you can leverage all of the benefits of DoH in your applications. However, you can also use the [`dns-packet`](https://github.com/mafintosh/dns-packet) library which has already gone through the trouble of abstracting everything we have talked about here in a succinct js module that can used directly in your website. 