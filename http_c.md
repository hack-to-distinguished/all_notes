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
4. [HTTP File Retrieval](#alejandro/HTTPFileRetrieval)
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


# chris/betterMsgReception

### Goal
Receive the message from the other client as soon as it's ready
### Problem 
When the client is in a send state (when he is typing a message) message reception is blocked. This is because the `fgets()` is a blocking function. The messages needs to be sent so that the client goes through another iteration of the loop for a `recv()` function to be called and display the incoming message.

### Solutions
- Make `fgets()` asynchronous
- Use `STDIN_FILENO()` 

# alejandro/HTTPFileRetrieval
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