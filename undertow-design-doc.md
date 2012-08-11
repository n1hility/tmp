Undertow Design Document
========================
Stuart Douglas <sdouglas@redhat.com>
v0.1, 2012

This is the design document for the undertow web server. It covers general
arcitecture and design considerations, it is not a requirements document.

Overview  
--------

The core Undertow architecture is based on the concept of lightweight async
handlers. These handlers are chained together to form a complete HTTP server.
The handlers can also hand off to blocking handlers backed by a thread pool.

This architecture is designed to give the end user complete flexibility when
configuring the server. For instance if the user simply wants to server up
static files, then they can confiure a server with only the handlers that are
required for that task. 

An example handler chain could be as follows:

![An Example Handler Chain](https://raw.github.com/stuartwdouglas/tmp/master/images/example.png "An Example Handler Chain")

Servlet functionality will be built on top of the async server core. The
servlet module will integrate with the core by providing its own handlers for
servlet specific functionality. As much as possible the servlet implementation
will use async handlers, only changing to blocking handlers when absolutly
required. This means that static resources packaged in a servlet will be
served up via async IO.


Core Server
===========

Incoming Requests
-----------------

Standard HTTP requests come into the server via the HTTPOpenListener, which
wraps the channel in a PushBackStreamChannel and  then hands off to the
HTTPReadListener. The HTTPReadListener parses the request as it comes in, and
once it has read all headers it creates a HTTPServerExchange and invokes the
root handler. Any of the content body / next request that is read by this
listener is pushed back onto the stream.

The HTTP parsing is done by a bytecode generated state machine, that
recognizes common headers and verbs. This means that parsing of common
headers can be done more quickly and with less memory usage, as if the header
value is known to the state machine an interned version of the string will be
returned directly, with no need to allocate a String or a StringBuilder. 

*NOTE:* It has not been shown yet if this will provide a significant
performance boost with a real workload to justify the extra complexity.

Other protocols such as HTTPS, AJP and SPDY etc support will be provided
through Channel implementation that as much as possible abstract away the
details of the underlying protocol to the handlers.

Handlers
--------

The basic handler interface is as follows:


	public interface HttpHandler {

	    /**
	     * Handle the request.
	     *
	     * @param exchange the HTTP request/response exchange
	     * @param completionHandler the completion handler
	     */
	    void handleRequest(HttpServerExchange exchange, HttpCompletionHandler completionHandler);
	}

The HttpServerExchange holds all current state to do with this request and the
response, including headers, response code, channels, etc. It can have
arbitrary attachments added to it, to allow handlers to attach objects to be
read by handlers later in the chain (for instance an Authentication handler
could attach the authenticated identity, which may then be used by a later
authorization handler to decide if the user should be able to access the
resource).







