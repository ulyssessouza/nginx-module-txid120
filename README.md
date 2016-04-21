# ``txid120``

If your application consists of many smaller parts, such as in a microservice architecture, it can be difficult to correlate requests made as the result of some end-user action.  For example, if your website combines data provided from many other services behind the scenes, one request from the user's browser can result in many requests to other parts of your network as it collects the information it needs to render the page.  It can be useful to know which of these requests are related to each other.  To do this, you can assign each _transaction_ (a request from an end user and all resulting internal requests) a __Transaction ID__.

Transaction IDs are unique strings that can be used to follow a chain of related requests or messages as they move through your network. This module generates these IDs.

## Setup

This is an Nginx module; it is meant to be used on an nginx proxy at the edge of your datacenter.  This module will fill a variable named ``$txid120`` for use in an Nginx directive like [``proxy_set_header``](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header). Set a header (maybe ``Txid``?) on every request entering your network.  If you use the same Nginx instance to also handle requests between services already within your network, don't reassign the header for those requests - the idea is that all requests related to the original end user request have the same Transaction ID.

In the simplest case, you can do something like this:

```
proxy_set_header Txid $txid120;
```

## Usage

Once your datacenter-edge Nginx is assigning Transaction IDs to new transactions, your internal services need to be modified to pass any incoming Transaction IDs along to sub-requests they make.  These are the general rules your services should follow:

* If a service **does** receive a Transaction ID, it should pass that same Transaction ID in any sub-requests it makes. (Use the same header you chose in the setup step above.)  It should also include the Transaction ID in any log messages it generates (so you can actually correlate events between services).
* If a service does **not** receive a Transaction ID, it should not send one (not even an empty string) to other services. It should use an empty string (or ``-`` or some other useful indication that it was missing) in its log messages where a Transaction ID would have gone.

If some of your services speak something other than HTTP internally, you can arrange a way to pass Transaction IDs between them on that protocol as well.  The goal is to be able to correlate all of the requests caused by a single initial request from an end user.

## Technical Details

The module is called ``txid120`` because it generates a 120-bit identifier.  (120 bits divides nicely into bytes and base64 encoding blocks, and so it is an efficient use of space; see the diagram below.) Services should treat the identifier as an opaque string, but it includes some useful properties.

The first 56 bits (7 bytes, or 9.33 characters in the encoded format) of the identifier represent a microsecond-granularity timestamp capable of representing times until the year 4147. The remaining bits are random. The identifier is encoded into 20 characters using a modified base64 alphabet which supports lexical sorting (0-9, ``:``, ``@``, A-Z, a-z).  Because the timestamp is at the front, the identifiers can be easily sorted into the order they were generated (and therefore the order in which the requests initially entered your network) using something like ``LANG=C sort list_of_txids.log``.

### Format Diagram

Each bit maps to the following meaning (``val``), bytes (``bin``), base64 characters (``b64``), and encoding blocks (``blk``):

```
c = seconds (used starting in year 2106), s = seconds (used now/soon), u = microseconds, r = random
val cccsssssssssssssssssssssssssssssssssuuuuuuuuuuuuuuuuuuuurrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrrr
bin 0.......1.......2.......3.......4.......5.......6.......7.......8.......9.......0.......1.......2.......3.......4.......
b64 0.....1.....2.....3.....0.....1.....2.....3.....0.....1.....2.....3.....0.....1.....2.....3.....0.....1.....2.....3.....
blk 0.......................1.......................2.......................3.......................4.......................
```

### Entropy Analysis

The remaining 64 bits are random data, which means there is a _very_ small probability of Transaction ID collisions within each microsecond. There can't be collisions across microseconds, since the microsecond-granularity time is included in the identifier.

At microsecond scale and assuming 10k reqs/sec, 64 bits of entropy gives a collision probability of:

```
bitprob(10000*(1/1000000), 64)
= 1 - 1 / Math.pow(Math.E, (Math.pow(10000*(1/1000000),2) / (2 * Math.pow(2,64))))
= 1 - 1 / ( e ^ (0.0001 / 3.68e19) )
= 1 - 1 / ( e ^ (2.717e-24) )
= 1 - 1 / ( 1 + 2.717e-24 )
= 2.717e-24
```

So, in any given microsecond, we're likely to see a collision 2.717e-24 of the time. (.000000000000000000000002717) Or, in other words, roughly one in every 3.86e23 microseconds will have a collision, or roughly once every 11.67 billion years, or roughly 1.2 times the expected lifetime of the Sun.
