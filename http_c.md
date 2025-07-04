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
4. [HTTP Text File Retrieval](#alejandro/HTTPTextFileRetrieval)
5. [HTTP Image File Retrieval](#alejandro/HTTPImageFileRetrieval)
6. [HTTP Head Method + Thread Pool](#alejandro/HEADMethodThreadPool)
# alejandro/HTTPResponse

recreate the error (on linux):
printf "get / HTTP/1.1\r\nHost: localhost\r\n\r\n" | nc 127.0.0.1 8080
printf "GET / HTTP/1.1\r\nHost: localhost" | nc 127.0.0.1 8080 
printf "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n" | nc 127.0.0.1 8080
```
bash -c '
HOST=127.0.0.1
PORT=8080
tests=(
  "❌ Lowercase Method" "get /data/geralt.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
  "❌ File does not exist" "GET /data/dean.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
  "❌ Missing CRLF after headers" "GET /data/geralt.txt HTTP/1.1\r\nHost: localhost"
  "✅ Proper GET Request" "GET /data/geralt.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
)
for ((i=0; i<${#tests[@]}; i+=2)); do
  echo -e "\n===== ${tests[i]} ====="
  printf "${tests[i+1]}" | nc $HOST $PORT
  echo -e "\n===========================\n"
  sleep 0.5
done'

```

think i know the error... Need to add null terminating characters at the end of each header name and header value extracted to avoid unnecessary characters being accidently added to the header value that is extracted. Need to also fix the error where when the 'Host' header is not at the first place in the header field

need to fix byte streaming issue... need the parser to separate packets
ERROR STUFF VALIDATION I NEED TO ADD:
```
printf "GET    /     HTTP/1.1\r\nHost: localhost\r\n\r\n" | nc 127.0.0.1 80 (EXTRA SPACES)
printf "GET / HTTP/1.1\r\n\r\n" | nc 127.0.0.1 80 (NO HOST)
printf "GET / HTTP/1.1\r\nHost: localhost" | nc 127.0.0.1 80 (?????)
```
chatgpt test code:
```
bash -c 'HOST=127.0.0.1; PORT=8080; tests=("✅ Valid Request" "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Missing Host Header" "GET / HTTP/1.1\r\n\r\n" "❌ Malformed Request Line" "GOT / HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Bad HTTP Version" "GET / HTTP/1.0.1\r\nHost: localhost\r\n\r\n" "❌ Extra Spaces in Request Line" "GET    /     HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Lowercase Method" "get / HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Missing CRLF after headers" "GET / HTTP/1.1\r\nHost: localhost" "❌ Improper Line Endings (LF)" "GET / HTTP/1.1\nHost: localhost\n\n" "❌ Header Injection Attempt" "GET / HTTP/1.1\r\nHost: localhost\r\nX-Test: test\r\nInjected: value\r\n\r\n" "❌ Non-ASCII Characters" "GET / HTTP/1.1\r\nHost: localhøst\r\n\r\n" "❌ Empty Request" "\r\n\r\n" "✅ Valid with Extra Headers" "GET / HTTP/1.1\r\nHost: localhost\r\nUser-Agent: TestClient/1.0\r\nAccept: */*\r\n\r\n"); for ((i=0; i<${#tests[@]}; i+=2)); do echo -e "\n===== ${tests[i]} ====="; printf "${tests[i+1]}" | nc $HOST $PORT; echo -e "\n===========================\n"; sleep 0.5; done'
```


asdf:
```
bash -c 'HOST=127.0.0.1; PORT=8080; tests=("✅ Valid Request" "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Missing Host Header" "GET / HTTP/1.1\r\n\r\n" "❌ Malformed Request Line" "GOT / HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Bad HTTP Version" "GET / HTTP/1.0.1\r\nHost: localhost\r\n\r\n" "❌ Extra Spaces in Request Line" "GET    /     HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Lowercase Method" "get / HTTP/1.1\r\nHost: localhost\r\n\r\n" "❌ Missing CRLF after headers" "GET / HTTP/1.1\r\nHost: localhost" "❌ Missing CRLF after headers" "GET / HTTP/1.1\r\nHost: localhost" "❌Improper Line Endings (LF)" "GET / HTTP/1.1\nHost: localhost\n\n" "❌ Header Injection Attempt" "GET / HTTP/1.1\r\nHost: localhost\r\nX-Test: test\r\nInjected: value\r\n\r\n" "❌ Non-ASCII Characters" "GET / HTTP/1.1\r\nHost: localhøst\r\n\r\n" "❌ Empty Request" "\r\n\r\n" "✅ Valid with Extra Headers" "GET / HTTP/1.1\r\nHost: localhost\r\nUser-Agent: TestClient/1.0\r\nAccept: */*\r\n\r\n"); for ((i=0; i<${#tests[@]}; i+=2)); do echo -e "\n===== ${tests[i]} ====="; printf "${tests[i+1]}" | nc $HOST $PORT; echo -e "\n===========================\n"; sleep 0.5; done'
```

GET 
/ 
HTTP/1.1\r\n Host: localhost\r\n\r\n

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


# alejandro/HTTPTextFileRetrieval
PLAN: 
- User can type URI in search bar through the web browser; e.g., let's say they want to 'GET' a picture of Lebron James from the web server, they would type in the browser '127.0.0.1:8080/lebron-james.jpg'
- Web server will need to validate and check if said file is on the server and respond accordingly:
	- If it is present, then send the file back.
	- Else send an error packet back to the user...
https://cdn.britannica.com/19/233519-050-F0604A51/LeBron-James-Los-Angeles-Lakers-Staples-Center-2019.jpg -> example URL

Problem encounter 1:
- Caused by entering 'http://127.0.0.1:8080/lebron-james.jpg' on one tab which returns a 200 OK packet and then 'http://127.0.0.1:8080/test.txt' which returns a 404 error packet... Both should have 200 OK packet.
lebron-james.jpg:
```
Bytes received: 474
HTTP Packet received from browser/client:
GET /lebron-james.jpg HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:139.0) Gecko/20100101 Firefox/139.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
```
test.txt:
```
Bytes received: 466
HTTP Packet received from browser/client:
GET /test.txt HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:139.0) Gecko/20100101 Firefox/139.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
0, i
```

test.txt by itself:
- Just making a single request with the test.txt URI works.
- Problem probably stems from the number of bytes being received is different...
```
Bytes received: 466
HTTP Packet received from browser/client:
GET /test.txt HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:139.0) Gecko/20100101 Firefox/139.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
```
The fix I am going to try is add null terminating character at the end of each packet...
- Doing this did fix it.

 10
P 80
r 114
i 105
o 111
r 114
i 105
t 116
y 121
: 58
  32
u 117
= 61
0 48
, 44
  32
i 105
 13

 10
 13

 10
---------
P 80
r 114
i 105
o 111
r 114
i 105
t 116
y 121
: 58
  32
u 117
= 61
0 48
, 44
  32
i 105
 13

 10
 13

 10
0 48
, 44
  32
i 105
 13

 10
 13

 10

- Finished checking if file exists on web server or not.
- Next steps -> send the requested file back to the user!!!!!!

Text File Request + Retrieval:
Open text file -> read file contents -> copy file contents in suitable HTTP packet where the 'Content-Type' is set to 'text/plain' -> send to client -> web browser renders said text file...
https://www.geeksforgeeks.org/c-program-to-read-contents-of-whole-file/
https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Messages#anatomy_of_an_http_message

test code:
```
bash -c '
HOST=127.0.0.1
PORT=8080
tests=(
  "❌ Lowercase Method" "get /data/geralt.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
  "❌ File does not exist" "GET /data/dean.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
  "❌ Missing CRLF after headers" "GET /data/geralt.txt HTTP/1.1\r\nHost: localhost"
  "✅ Proper GET Request" "GET /data/geralt.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
)
for ((i=0; i<${#tests[@]}; i+=2)); do
  echo -e "\n===== ${tests[i]} ====="
  printf "${tests[i+1]}" | nc $HOST $PORT
  echo -e "\n===========================\n"
  sleep 0.5
done'

```

# alejandro/HTTPImageFileRetrieval
GOALS:
- Get .jpg and .jpeg file transfer and request working...

Test code:
```
bash -c '
HOST=127.0.0.1
PORT=8080
tests=(
  "❌ Lowercase Method" "get /data/geralt.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
  "❌ File does not exist" "GET /data/dean.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
  "❌ Missing CRLF after headers" "GET /data/geralt.txt HTTP/1.1\r\nHost: localhost"
  "✅ Proper GET Request with .txt file" "GET /data/geralt.txt HTTP/1.1\r\nHost: localhost\r\n\r\n"
  "✅ Proper GET Request with .jpg image" "GET /data/geralt.jpg HTTP/1.1\r\nHost: localhost\r\n\r\n"
)
for ((i=0; i<${#tests[@]}; i+=2)); do
  echo -e "\n===== ${tests[i]} ====="
  printf "${tests[i+1]}" | nc $HOST $PORT
  echo -e "\n===========================\n"
  sleep 0.5
done'
```

https://stackoverflow.com/questions/23714383/what-are-all-the-possible-values-for-http-content-type-header
-> `image/jpeg`, content-type header value...
don't need encoding... -> can think about it later.
Need to figure out how to encode a .jpg/.jpeg file into the message body of the HTTP packet:
- Need to specify content codings -> 'Content codings are primarily
   used to allow a document to be compressed or otherwise usefully
   transformed without losing the identity of its underlying media type
   and without loss of information. Frequently, the entity is stored in
   coded form, transmitted directly, and only decoded by the recipient.'
	- https://www.rfc-editor.org/rfc/rfc2616#section-3.5 (3.5 Content Codings)

Process for image retrieval:
1) Binary read the data of the image.
2) Get the length of the image data and malloc a buffer of enough size.
3) Then, I need to copy said data into an appropriate buffer of enough size (need to also think about error handling, if the file is too big...).
4) Format both length and raw binary data into an appropriate HTTP packet.
5) Send the packet over to the client (web browser).

https://www.man7.org/linux/man-pages/man2/stat.2.html
https://www.man7.org/linux/man-pages/man3/stat.3type.html
https://pubs.opengroup.org/onlinepubs/7908799/xsh/sysstat.h.html
# alejandro/HEADMethodThreadPool
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods/HEAD
HEAD method is basically a GET method, but only retrieves the header fields of a specific resource.
-> ```
```
curl -I http://localhost:8080
``` 
test with this command...

FOCUS Thread Pool
Main Components:
- Circular Queue for the tasks:
	- FUNCTION 1 -> Enqueue tasks (add task for worker thread(s)).
	- FUNCTION 2 -> Dequeue tasks (get task for worker thread).
	- FUNCTION 3 -> Execute Tasks by using and waking a worker thread.
- Array of worker threads (the actual thread pool).
- The task will just be my function the executes the HTTP parsing -> will require a struct for custom data required.

STRUCTS Required:
- Circular Queue struct (thread pool struct).
- HTTP struct.

REFACTOR:
http.h
http.c
threadpool.c
threadpool.h

HEAD Method stuff:
- END_OF_HEADERS -> send_requested_HEAD
	- send_requested_HEAD should facilitate all the HEAD requests...

https://stackoverflow.com/questions/26306644/how-to-display-st-atime-and-st-mtime
https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Last-Modified
https://stackoverflow.com/questions/4217037/catch-ctrl-c-in-c
https://thelinuxcode.com/signal_handlers_c_programming_language/

- always mutex lock for shared resources
- cond signal wakes one thread up, cond broadcast wakes all threads up.
	- cond_wait must be used in conjunction with a predicate inside a while loop
# chris/betterMsgReception

### Goal
Receive the message from the other client as soon as it's ready
### Problem 
When the client is in a send state (when he is typing a message) message reception is blocked. This is because the `fgets()` is a blocking function. The messages needs to be sent so that the client goes through another iteration of the loop for a `recv()` function to be called and display the incoming message.

### Solutions
- Use `STDIN_FILENO()` 
# chris/msgUI - chris/msgFromBrowser

### Goal
Build a user interface for the messaging portion of HTTPC. Build a web view using JavaScript. 
Build a command line user interface where people can SSH into HTTC to send a receive messages straight from the CLI.
The next step would be building a GUI for desktop in C.

Web view for the chat
- Build a view that would show some floating text. Like something that's not supposed to be answered. Just messages that float into the void. They go from right to left and can appear at any y-axis level.
SSH through the CLI for the chat. 

All messages should be saved in a database at some point. For now just save them in the current user session.


### Problem 
The browser can't directly execute C code

### Solutions
Option 1: 
Create a proxy (maybe in Node.ts) that would become client.c (replacing it) for the browser side of things. And that sever side functionality would convert the HTTP requests sent by the React app into raw TCP packets to the C server. The proxy would also convert TCP packets into HTTP back to React

| Layer        | Purpose                            |
| ------------ | ---------------------------------- |
| C server     | TCP server handling message logic  |
| Proxy (Node) | Converts HTTP to raw TCP           |
| React        | Sends HTTP requests (from browser) |
Option 2:
Make the server handle HTTP requests instead of just raw TCP packets


Option 2 is the way to go imo.
For now, just focus on finding a way to communicate with the C code.

#### Notes
Since the server just instantly sends messages to whoever is connected, you can connect and receive all the message that were sent before. You might need to be constantly pinging the server in order to get any messages.

On the frontend I am trying to fetch data from the C server but the C server is pushing data. I should just be waiting for data to arrive. I shouldn't pull, I should just be open to receive.

2025-06-22:
The interval check is connecting and disconnecting from the server. Not maintaining the connection. That means that I always get the "connected to the sever" msg and not the messages sent form other people. I need to open a connection and maintain it.


#### Display
The MessageBox page will call the C API and here we handle the displaying of each message.
They should be displayed from bottom to top as if coming from below the screen.
They slowly float to the top, they aren't forever, only available for a short period of time.
If no messages are sent for a while, the most recent messages (maybe 10 max) float back down and wait. When a messages is sent, they're pushed back up.
The web view of HTTPC is more for fun (this one at least). The actual important messaging section with extra security and stuff will be through a different medium. I'm thinking something SSH related. Accessible via the command line or something else that would be easily portable to a mobile device. All security will be handled on the sever side (C) so doing the web client this way doesn't weaken the important part

Managed using the long polling
### Sources:
[C in react using web assembly](https://dev.to/iprosk/cc-code-in-react-using-webassembly-7ka)
[C++ in react](https://medium.com/@stormbee/supercharge-your-react-app-with-c-e89025f03b37)
[Long Polling](https://medium.com/@ignatovich.dm/implementing-long-polling-with-express-and-react-2cb965203128) - Used this.
