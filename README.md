# Address Normalization Microservice

## Abstract

Addresses come into the system as user-entered details.  Users are bad at entering details.  For instance, we might see something like:

```js
{
    "address1" : "123 \"main street\" overland park ks",
    "address2" : "",
    "city" : "lenexa",
    "state" : "mo",
    "zip" : "",
    "zip4" : ""
}
```

There are several problems with this address, and they mostly stem from the user's input.  There are quotes that shouldn't be there, the (incorrect) city is included in the address line, as well as the correct state.  The actual "state" field is incorrect, and there's no ZIP information.  As it stands, this address does not qualify for the reduced postage that USPS offers if the address is pre-normalized, so it must go through a sanitzation process.  This is handled by 3rd party APIs.

## Third party APIs

Currently, we are using the normalization services provided by LOB.com.  They expose an API that will take one or more addresses, sanitize them, and return the corrected details along with a confidence score from 0-100.  Any addresses that are not able to be normalized are indicated in the response with empty fields, and the confidence is 0.  The address above might get normalized to something like:

```js
{
    "address1" : "123 MAIN ST",
    "address2" : "",
    "city" : "SHAWNEE",
    "state" : "KS",
    "zip" : "66206",
    "zip4" : "4802"
}
```

From here, this address is considered "valid" and will qualify for the reduced postage rate.  The downside to using these third party APIs is that they have different mechanisms for calling, have API rate limits, do not offer pre-validation or caching, and more.  The first restriction, "different mechanisms for calling" is the most troublesome, because this means multiple applications will have to be updated and re-tested if we change providers.  By wrapping the process in a microservice and updating applications to call that instead, the upstream provider can be changed far more easily and with much less testing.  It also gives us a single spot in which to apply improvements such as caching or rate limiting.

## Requirements

### One-shot

Several applications need to normalize a single address quickly, such as responding to another API call with a pass/fail.  The normalization service must account for this use case and respond quickly.

### Batching

A much more common use-case is sanitizing an entire file's worth of addresses in one shot.  This would be accomplished by passing in an array of unsanitized addresses as a JSON payload, and receiving a reply a short (synchronous) time later.  This call is necessarily expected to be slower than the one-shot use case due to the increased data, but it does not necessarily have to be a separate endpoint.

### API structure

Requests should come in as JSON via POST to a fixed endpoint.  Authentication is handled via API keys in a custom HTTP header.  The response is roughly the same format.

*request*:

```plain
POST /api/normalize
X-Api-Key: 031cfe2e6dfa40799fd80ed8e9011484

[
    {
        "address1" : string (1-50 chars, utf8, required),
        "address2" : string (1-50 chars, utf8, optional),
        "city" : string (1-80 chars, utf8, required [1]),
        "state" : string (1-50 chars, utf8, required [1]),
        "zip" : string (1-10 chars, numeric, required [2]),
        "zip4" : string (4 chars, numeric, optional [2]),
        "tag" : string (1-64 chars, utf8, optional)
    },
    ... repeat if more are needed
]
```

*response*:

```plain
X-Api-Rate-Remaining: 280
X-Api-Rate-Window: 3600

[
    {
        "address1" : string (1-50 chars, utf8),
        "address2" : string (1-50 chars, utf8),
        "city" : string (1-80 chars, utf8),
        "state" : string (2 chars, maps to short state code),
        "zip" : string (5 chars, numeric),
        "zip4" : string (4 chars, numeric),
        "tag" : string (1-64 chars, unmodified from request),
        "confidence" : number (0-100),
        "valid" : bool
    },
    ... repeats as needed
]
```

In the request payload, a combination of city+state OR zip must be provided.  Both can be provided.  The zip4 is not required, but should be included if available in the source data.  The "tag" property is used to associate each individual payload from the request to the response, and is passed through unmodified.  This will commonly be a random value, such as a GUID.  It must not be manipulated in any way.  The response is not required to be in the same order as the request.  In the case of an unmappable address, the response object is still provided with the original tag, but the address fields will all be empty.  The "confidence" will be 0, and "valid" will be false.  Invalid addresses may appear anywhere in the response and should not error out the entire request.

### Future optimizations

No particular order or priority.  **TODO: Expand more later**

* pre-sanitize inputs to a more canonical format
* reject addresses that are trivially simple to determine if invalid
* use hashing to track previously-seen addresses
* store sanitized hashes and their responses to skip lookups we've already performed in the past
* allow multiple backend providers and switch between them as needed
* enqueue unrelated requests to maximize usage of upstream batching without exceeding API rate limits
