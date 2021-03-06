---
layout: post
title: "How does curl post data?"
date: 2013-08-15
preview: a slightly closer look at a standard Unix curl program
---

<code>curl</code> is the Unix library for sending and receiving data from the internet (I'm sure it does much more than that, but I'm not equipped to talk about it!). With <code>curl</code>, you can post data with the <code>-d</code> flag. But what does that do? Disclaimer: This is more of a demonstration than a full tutorial.

Let's say we're issuing a query to ElasticSearch, like so:

    $ curl -XGET http://localhost:9200/articles/_search -d '{"query":{"query_string":{"query":"cat"}}}'

we can examine how curl is interacting with the specified url by appending the above command with a trace, like so:

     --trace-ascii /dev/stdout

[I learned this trick here](http://superuser.com/questions/291424/how-do-you-display-post-data-with-curl). The output from this command would then be:

    == Info: About to connect() to localhost port 9200 (#0)
    == Info:   Trying ::1...
    == Info: Connection refused
    == Info:   Trying 127.0.0.1...
    == Info: connected
    == Info: Connected to localhost (127.0.0.1) port 9200 (#0)
    => Send header, 230 bytes (0xe6)
    0000: GET /articles/_search HTTP/1.1
    0020: User-Agent: curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0
    0060:  OpenSSL/0.9.8r zlib/1.2.5
    007c: Host: localhost:9200
    0092: Accept: */*
    009f: Content-Length: 42
    00b3: Content-Type: application/x-www-form-urlencoded
    00e4:
    => Send data, 42 bytes (0x2a)
    0000: {"query":{"query_string":{"query":"cat"}}}
    == Info: upload completely sent off: 42 out of 42 bytes
    <= Recv header, 17 bytes (0x11)
    0000: HTTP/1.1 200 OK
    <= Recv header, 47 bytes (0x2f)
    0000: Content-Type: application/json; charset=UTF-8
    <= Recv header, 21 bytes (0x15)
    0000: Content-Length: 122
    <= Recv header, 2 bytes (0x2)
    0000:
    <= Recv data, 122 bytes (0x7a)
    0000: {"took":0,"timed_out":false,"_shards":{"total":5,"successful":5,
    0040: "failed":0},"hits":{"total":0,"max_score":null,"hits":[]}}
    == Info: Connection #0 to host localhost left intact
    {"took":0,"timed_out":false,"_shards":{"total":5,"successful":5,"failed":0},"hits":{"total":0,"max_score":null,"hits":[]}}
    == Info: Closing connection #0

We see the communication between our machine via <code>curl</code> and the ElasticSearch server follow these steps:

1. <code>curl</code> sends a header to the specified uri to open the connection. ElasticSearch accepts this header and opens up the connection.
2. The data packet is sent through the open connection.
3. ElasticSearch sends its response header with a <code>HTTP/1.1 200</code> success status code.
4. ElasticSearch sends the data, which our machine accepts and closes the connection.

That pattern illustrates what happens during a typical data post via curl, or any other http client for that matter.

Now for fun, let's contrast this correct usage of the ElasticSearch api with something less useful

    $ curl -X GET http://localhost:9200/articles/_search?%7B%22query%22%3A%7B%22query_string%22%3A%7B%22query%22%3A%22cat%22%7D%7D%7D  --trace-ascii /dev/stdout
    == Info: About to connect() to localhost port 9200 (#0)
    == Info:   Trying ::1...
    == Info: Connection refused
    == Info:   Trying 127.0.0.1...
    == Info: connected
    == Info: Connected to localhost (127.0.0.1) port 9200 (#0)
    => Send header, 238 bytes (0xee)
    0000: GET /articles/_search?%7B%22query%22%3A%7B%22query_string%22%3A%
    0040: 7B%22query%22%3A%22cat%22%7D%7D%7D HTTP/1.1
    006d: User-Agent: curl/7.24.0 (x86_64-apple-darwin12.0) libcurl/7.24.0
    00ad:  OpenSSL/0.9.8r zlib/1.2.5
    00c9: Host: localhost:9200
    00df: Accept: */*
    00ec:
    <= Recv header, 17 bytes (0x11)
    0000: HTTP/1.1 200 OK
    <= Recv header, 47 bytes (0x2f)
    0000: Content-Type: application/json; charset=UTF-8
    <= Recv header, 23 bytes (0x17)
    0000: Content-Length: 35894
    <= Recv header, 2 bytes (0x2)
    0000:
    <= Recv data, 16295 bytes (0x3fa7)
    0000: {"took":39,"timed_out":false,"_shards":{"total":5,"successful":5
    0040: ,"failed":0},"hits":{"total":13,"max_score":1.0,"hits":[{"_index
    0080: ":"articles","_type":"article","_id":"4","_score":1.0, "_source"
    00c0:  : {"type":"article","id":4,"summary":"<p>With grep, you can not
    0100:  only return the matching line for your string or regex, but als

and a few hundred lines later

    == Info: Closing connection #0

What happened? We didn't *send* any data, we just requested a uri (namely <code>http://localhost:9200/articles/_search</code>) with a uri encoded string concatenated at the end. That's *not* a proper POST and *not* how the ElasticSearch api handles a query. Hence, the server defaulted by querying all, returning every document (with a default limit of ten documents) in the response body. We sent an ok get, but the params weren't something the server could work with; the server wanted actual POST data as in the first example above.
