---
tags:
  - HTTP
  - SWE
  - C_Programming
aliases:
  - Hyper Text Transfer Protocol
Associated: "[[tank_squared]]"
---

# Table of Contents
1. [HTTP Response](#alejandro/HTTPResponse)
2. [Multi-client messaging](#chris/redirectMsg)
3. [[#chris/betterMsgReception|Instant message reception]]
# alejandro/HTTPResponse

stateful parsing (idea from deepseek; used knowledge from state machines from CS)

- end of headers is the final accepting state if parsing is successful.
- error_state is a dead state → send error http packet back.

![[packet_diagram.png]]
echo -e "GET /\r\nHost: localhost\r\n\r\n" | nc 127.0.0.1 80 → use this for testing (netcat)

Example GET HTTP Request Packet from Browser:

```html
GET / HTTP/1.1
Host: 127.0.0.1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:137.0) Gecko/20100101 Firefox/137.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: cross-site
Priority: u=0, i
ty: u=0, i
```

```html
GET / HTTP/1.1\\r\\n  
Host: example.com\\r\\n  
Content-Type: text/html\\r\\n  
\\r\\n  

html block with crlf
```

- Need to create a parse function that will check the status line of packets received from the browser/client.
    - Gonna try and parse the status line first, and then just print it out.

### Basic Parsing Stuff I can do (from chatgpt)

1. **Request Line Parsing (DONE, except for URI)**
    
    - Extract the HTTP method (GET, POST, etc.)
    - Extract the URI (e.g., `/index.html`)
    - Extract the HTTP version (e.g., `HTTP/1.1`)
2. **Header Fields Parsing**
    
    Parse headers into key-value pairs:
    
    - `Host:` → for virtual hosting or logging
    - `User-Agent:` → for logging or conditional responses
    - `Accept:` → to serve different content types (basic content negotiation)
    - `Connection:` → check for `keep-alive` vs `close`
3. **Simple Error Handling (slightly done)**
    
    - Return `400 Bad Request` if the request is malformed
    - Return `501 Not Implemented` if the method is unsupported

[http://127.0.0.1/](http://127.0.0.1/) (made it bind default to port 80)

(CRLF = Carriage Return + Line Feed. ‘\r\n’)

- Need to create a HTTP response packet. I am just going to focus on responding with a response packet; no parsing will be implemented yet.
    - Split (simple) HTTP response packet into two:
        - Header.
            - Status Line.
                - ABNF Syntax: `"HTTP/" 1*DIGIT "." 1*DIGIT SP 3DIGIT SP`
            - Entity Header Field.
                - Gonna focus on two fields: ‘Content-Type’ and ‘Content-Length’ `"Content-Length" ":" 1*DIGIT` `"Content-Type" ":" media-type`
        - Body.

Different status codes:
![[status_codes.png]]

- Make parse method better (might just redo it, as it is trash; need to find good examples):
    - Modularise…
    - Focus on detecting whether header is malformed or not:
        - Need to check every single header field.
        - Need to check that ends properly with double CRLF.
        - Need to also check that each field is separated by a single CRLF (’\r\n\r\n’).
    - Need to also stream bytes better… So when there are two http packets send one after the other, I should be able to only parse through one of them at a time (e.g., I need a delimmiter or something to only store data relevant to the current http packet to be parsed so the other one doesn’t also get stored within said buffer from byte stream.)
        - update receive http method.

### Sources Used
[What's the right way to parse HTTP packets?](https://stackoverflow.com/questions/17460819/whats-the-right-way-to-parse-http-packets)

[RFC 2616: Hypertext Transfer Protocol -- HTTP/1.1](https://www.rfc-editor.org/rfc/rfc2616)

[RFC 822: STANDARD FOR THE FORMAT OF ARPA INTERNET TEXT MESSAGES](https://datatracker.ietf.org/doc/html/rfc822#section-4)

[RFC 1945: Hypertext Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc1945#page-21)

[example polliing server](https://github.com/64/hh/tree/master)

[associated reddit thread](https://www.reddit.com/r/C_Programming/comments/7bnscf/multithreaded_epoll_server_design/)

# chris/redirectMsg
Handling multiple connections:
- The initial fd you get using sockfd can use listen to keep track of a backlog of items but you can only accept a connection from one at a time. That's why the `accept()` function doesn't have the `BACKLOG` parameter. 
  To handle multiple connections, I can think of 2 options:
	- Use the threaded server NJ built
	- Send the client to the back of the queue after you receive a message from it. (going to explore this idea)


### Handling multiple connections on a single thread:
##### **Server:**
- In a WHILE loop (maybe a while server fd is available loop), 
	- listen for client connections.
    - IF anyone is attempting to connect:
	    - Add the connection to a list/queue of fd (Use the pollfd struct).
    - Open a FOR loop that cycles through the list and 
	    - Checks each fd to see if it’s attempting to send something (POLLOUT I think) - IF yes:
	        - Receive that message
	        - Store the message
    - Open a FOR loop and cycle through the fd list:
        - Check if any are ready to receive (POLLIN) - IF yes:
		    - Send the stored message.
	- *IF NEEDED:* FOR loop through the clients:
		- Check if they've disconnected.
		- If the client is disconnected, remove it from the list.

*To add later for improvements:*
- Don't limit the amount of clients. Once the limit is reached, increase the size

##### **Client**:
- Create a pollfd for the server fd. 
- In a WHILE loop
	- Check IF server is trying to send, (POLLOUT) - IF yes:
		- Receive the message and print it on screen
	- Check IF server is ready to receive (POLLIN) - IF yes:
		- Create the message sending mechanism
		- Send the message entered by the user


# chris/betterMsgReception

### Goal
Receive the message from the other client as soon as it's ready
### Problem 
When the client is in a send state (when he is typing a message) message reception is blocked. This is because the `fgets()` is a blocking function. The messages needs to be sent so that the client goes through another iteration of the loop for a `recv()` function to be called and display the incoming message.

### Solutions
- Make `fgets()` asynchronous
- Use `STDIN_FILENO()` 


# chris/msgUI

### Goal
Build a user interface for the messaging portion of HTTPC. Build a web view using JavaScript. 
The next step would be building a GUI for desktop in C.

### Problem 


### Solutions