# Web Server FAQ

<!-- ============================================================================= -->

<p><details><summary><b>Why do we have to specify the <tt>Host</tt> field in the HTTP request? Isn't it redundant since we just connected to that host?</b></summary><p>

In the good old days, there was typically only one host per IP address, and
vice-versa. That is, if we connected to IP address, `198.51.100.20`, there would
have only been one host (e.g. `www.example.com`) associated with it.

In that case, the `Host` field would truly be redundant. (And in fact, it was
not included in the original 1.0 version of HTTP.)

But since then, we've moved to a time where a single computer with a single IP
might host hundreds or even thousands of different websites.

In such a case, the client needs to tell the server not only the file it's
interested in (e.g. `/funnycat.gif`), but also the host that it's coming from
(e.g. `cats.example.com`). After all, there might be hundreds of domains on this
server, and hundreds of `/funnycat.gif` files.

So the `Host` field becomes necessary. The client not only needs to specify the
file they're interested in, but also the domain they're expecting to find it on.

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>In <tt>send_response()</tt>, the <tt>body</tt> argument is passed as a <tt>void*</tt>. Is there some scenario where the <tt>body</tt> is not a string?</b></summary><p>

In the MVP, no it's always a string.

But if you want to serve any kind of binary file (e.g. `foo.jpg`), you'll be
`send()`ing data that's _not_ a string.

This is why `send_response()` takes a `void*`--it can point to any type of data,
string or not.

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>I rebuilt and re-ran the server, but nothing's happening.</b></summary><p>

You have to hit it with a web request. Either send your browser to
`http://localhost:3490/`, or run curl:

```shell
curl -D - http://localhost:3490/
```

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>Why is the <tt>Content-Length</tt> different than the number of bytes we pass to <tt>send()</tt>?</b></summary><p>

It's because they refer to the size of the _payload_ at different layers in the
protocol stack.

Remember that HTTP runs on top of TCP, and TCP runs on top of IP, and IP runs on
top of Ethernet (on your LAN, anyway).

So we first put together our HTTP packet, and in there we put the size of the
HTTP data in `Content-Length`, and doesn't count the HTTP header.

But then we wrap _that entire thing_ inside TCP. (Well, the OS does it for us
when we call `send()`.) So the entirety of the HTTP data, header and body, needs
to be wrapped up in TCP and sent. So when we call `send()`, we give it that
entire length.

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>Does the order of the fields in the HTTP header matter?</b></summary><p>

No. The key-value pairs can be in any order.

Caveat: the first line of the header is always something like this for requests:

```http
GET /index.html HTTP/1.1
```

and the first line of the response is always something like this:

```http
HTTP/1.1 200 OK
```

But _after_ that, with all the things like `Content-Length` and `Content-Type` and all that, those can be in any order.

Don't forget to end your header with an empty line!

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>After sending the request and getting the response, my client is just sitting there, waiting, and not closing the connection.</b></summary><p>

Make sure you have the

```http
Connection: close
```

field in your header.

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>After sending a <tt>GET</tt> request, the server is just sitting there waiting for something and not responding.</b></summary><p>

Make sure you end your request header with a blank line. That's how the server
knows the header is complete.

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>How can I use <tt>curl</tt> on the command line to test my server?</b></summary><p>

`curl` sends web requests from the command line and prints the results on
standard output.

Just hit the URL and print the body:

```shell
curl http://localhost:3490/
```

Hit the URL and print out the response headers and the body:

```shell
curl -D - http://localhost:3490/
```

Hit the URL, print out the request headers (and body), print out the response
headers and body:

```shell
curl -v http://localhost:3490/
```

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>How do browsers handle caches?</b></summary><p>

The browser-side cache (as opposed to the server-side cache that you'll be
writing) has the goal of speeding web page loads by reducing network traffic.

When loading a web page (or other piece of data), the browser first looks in its
cache on disk to see if the data is there. If it is, it can display it
immediately and the user doesn't have to wait for it to come in over the
network.

Of course, there are more details that need to be hashed out.

* Does the browser have a limit on the number of items in the cache?
* Does the browser have a limit on the amount of data in the cache?
* How does the browser refresh the cache or expire elements in the cache?
* How does the browser know which pages should and should not be cached?

Most browsers allow you to see a limit on the cache size in bytes. Old
information that exceeds that limit will be discarded.

Of course, when the browser hits the cache but finds some old data, it needs to
_refresh_ that cache entry from the server again.

Entries might be old, or maybe a web page has said it should never be cached.

A web server can offer hints to a web browser about how data should be cached.
For more information, see the [Cache-Control
header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control).

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>Where can I find a complete list of MIME types?</b></summary><p>

Since people are adding new MIME types all the time, there's not really a such
thing as a _complete_ list.

(You can make up your own MIME types. If they're not official, the second part
of the MIME type should be prefixed with `x-`, like `application/x-bzip`.)

Common MIME types can be found at [MDN's incomplete list of MIME
types](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types/Complete_list_of_MIME_types).

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>Our server only sends out one file at a time. How do browsers get multiple files, like if <tt>index.html</tt> refers to <tt>styles.css</tt>?</b></summary><p>

Turns out they do them in separate requests.

First the browser will get `index.html`. Then it parses the HTML and sees that
it has references to, for example, `styles.css` and `funnycats.gif`. It then
issues two more requests, one for each of those files.

There's a fair amount of overhead to this, so modern browsers typically make a
single TCP connection over the network, and then transmit multiple HTTP requests
over it. Conceptually, however, this still sends multiple HTTP requests, just
like above.

In our server, we specify the header

```http
Connection: close
```

which tells the browser we're hanging up after this one response. This is just
to make our code easier.

</p></details></p>

<!-- ============================================================================= -->

<p><details><summary><b>How can I fix my server to return binary data as well as text data? I want to get <tt>cat.png</tt> back to the browser.</b></summary><p>

If you are building your HTTP response, both header and body, with a single
`sprintf()`, you'll have trouble getting the body in there. This is because
`sprintf()` with `%s` prints a string, and a string ends on the first `'\0'`
character. Undoubtedly the PNG image is full of `0`s, so it stops printing short
of end of the data.

There are two options here:

1. Use two calls to `send()`.

   1. Use `sprintf()` to make the header, but that's all.

   2. Call `send()` once to send the header.

   3. Call `send()` again to send the body.

2. Use `memcpy()` to append the body after the header.

   1. Use `sprintf()` to make the header, but that's all.

   2. Use `memcpy()` to copy the body just after the end of the header.
      `memcpy()` copies a specified number of bytes; it doesn't stop at the
      first `0` byte.

   3. Call `send()` to send out the complete HTTP response.

</p></details></p>


<!--
TODO:

-->

<!-- ============================================================================= -->

<!--
Template:

<p><details><summary><b></b></summary><p>
</p></details></p>
-->
