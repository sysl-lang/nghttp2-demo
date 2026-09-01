# nghttp2-demo

**HTTP/2 over TLS, both ends in one process, with no socket anywhere.**

The worked example for [`sh.sysl.nghttp2`](https://github.com/sysl-lang/nghttp2), and the thing it
actually demonstrates is the *seam*: `sh.sysl.openssl` and `sh.sysl.nghttp2` were written a
repository apart and neither names the other. What they have in common is that neither owns a file
descriptor — one turns bytes into ciphertext and back, the other turns bytes into HTTP/2 frames and
back — so composing them is four buffers and a loop.

```
plaintext  --> TLS stream --> ciphertext --> the peer's TLS stream --> plaintext
   ^                                                                      |
   +------------------------- HTTP/2 session <----------------------------+
```

```
sysl build .
./nghttp2-demo
```

```
TLS 1.3, ALPN and HTTP/2, with no socket in the program

handshake done
ALPN chose h2 at the client, h2 at the server
nghttp2 agrees the list offers h2: true

connection open
  server: the peer's settings arrived
  client: the peer's settings arrived

GET /hello on stream 1
  server: stream 1 headers -- :method: GET, :scheme: https, :path: /hello, :authority: localhost, user-agent: sysl-nghttp2-demo
  server: stream 1 ended

the answer, decrypted and unframed:
  client: stream 1 headers -- :status: 200, content-type: text/plain, content-length: 33
  client: stream 1 body -- Hello from a sysl HTTP/2 server.

  client: stream 1 ended
  client: stream 1 closed

done
```

## Nothing here is a mock

The certificate is a real self-signed EC certificate for `localhost`, the handshake is a real TLS
1.3 handshake, and the twenty-four bytes of HTTP/2's connection preface really do go through the
cipher. What is missing is only the part that would need a port: `pump` is the socket, and it is
nine lines.

**ALPN is the whole reason this composition had to wait.** There is no other way to reach HTTP/2
over TLS — no upgrade header, no version marker, nothing in the request itself. A server that cannot
select a protocol during the handshake cannot serve `h2` at all, whatever else it can do. `openssl`
0.2.0 added the server method and `SSL_CTX_set_alpn_select_cb`; this program is what that unblocked.

**A composition that needs a port is a composition nobody can check on a train**, which is why the
demo runs in memory rather than binding one. The line where a real deployment differs is `pump`, and
neither library above changes at all.

## What it needs

OpenSSL and nghttp2, both from the system:

```
brew install openssl@3 nghttp2
sudo apt install libssl-dev libnghttp2-dev
sudo pacman -S openssl libnghttp2
```
