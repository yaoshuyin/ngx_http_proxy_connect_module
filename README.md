```bash
$ wget http://nginx.org/download/nginx-1.18.0.tar.gz
$ wget https://github.com/yaoshuyin/ngx_http_proxy_connect_module/archive/v0.0.1.tar.gz

$ tar -xzvf nginx-1.18.0.tar.gz
$ tar -xvf v0.0.1.tar.gz

$ cd ngx_http_proxy_connect_module-0.0.1
$ 下载 https://github.com/yaoshuyin/ngx_http_proxy_connect_module/blob/master/patch/proxy_connect_rewrite_1018.patch 的内容，存入patch/proxy_connect_rewrite_1018.patch

$ cd ../nginx-1.18.0/
$ patch -p1 < ../ngx_http_proxy_connect_module-0.0.1/patch/proxy_connect_rewrite_1018.patch

$ ./configure --prefix=/opt/nginx_proxy --with-http_ssl_module --add-module=../ngx_http_proxy_connect_module-0.0.1/
$ make && make install
```

cat /opt/nginx_proxy/conf/nginx.conf
```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
       listen 8888;
    
       #resolver 8.8.8.8;
       resolver 114.114.114.114;
       proxy_connect;
       proxy_connect_allow            80 443 563;
       proxy_connect_connect_timeout  10s;
       proxy_connect_read_timeout     10s;
       proxy_connect_send_timeout     10s;
     
       location / {
          proxy_pass $scheme://$http_host$request_uri;
       }
    }
}
```

```bash
$ /opt/nginx_proxy/sbin/nginx
$ curl -x localhost:8888 https://www.baidu.com
```

..............................................................
..............................................................


name
====

This module provides support for [the CONNECT method request](https://tools.ietf.org/html/rfc7231#section-4.3.6).
This method is mainly used to [tunnel SSL requests](https://en.wikipedia.org/wiki/HTTP_tunnel#HTTP_CONNECT_tunneling) through proxy servers.

Table of Contents
=================

   * [name](#name)
   * [Example](#example)
      * [configuration example](#configuration-example)
      * [example for curl](#example-for-curl)
      * [example for browser](#example-for-browser)
      * [example for basic authentication](#example-for-basic-authentication)
   * [Install](#install)
      * [select patch](#select-patch)
      * [build nginx](#build-nginx)
         * [build as a dynamic module](#build-as-a-dynamic-module)
      * [build OpenResty](#build-openresty)
   * [Test Suite](#test-suite)
   * [Error Log](#error-log)
   * [Directive](#directive)
      * [proxy_connect](#proxy_connect)
      * [proxy_connect_allow](#proxy_connect_allow)
      * [proxy_connect_connect_timeout](#proxy_connect_connect_timeout)
      * [proxy_connect_read_timeout](#proxy_connect_read_timeout)
      * [proxy_connect_send_timeout](#proxy_connect_send_timeout)
      * [proxy_connect_address](#proxy_connect_address)
      * [proxy_connect_bind](#proxy_connect_bind)
      * [proxy_connect_response](#proxy_connect_response)
   * [Variables](#variables)
      * [$connect_host](#connect_host)
      * [$connect_port](#connect_port)
      * [$connect_addr](#connect_addr)
      * [$proxy_connect_connect_timeout](#proxy_connect_connect_timeout-1)
      * [$proxy_connect_read_timeout](#proxy_connect_read_timeout-1)
      * [$proxy_connect_send_timeout](#proxy_connect_send_timeout-1)
      * [$proxy_connect_resolve_time](#proxy_connect_resolve_time)
      * [$proxy_connect_connect_time](#proxy_connect_connect_time)
      * [$proxy_connect_response](#proxy_connect_response-1)
   * [Compatibility](#compatibility)
      * [Nginx Compatibility](#nginx-compatibility)
      * [OpenResty Compatibility](#openresty-compatibility)
      * [Tengine Compatibility](#tengine-compatibility)
   * [FAQ](#faq)
   * [Known Issues](#known-issues)
   * [See Also](#see-also)
   * [Author](#author)
   * [License](#license)

Example
=======

Configuration Example
---------------------

```nginx
 server {
     listen                         3128;

     # dns resolver used by forward proxying
     resolver                       8.8.8.8;

     # forward proxy for CONNECT request
     proxy_connect;
     proxy_connect_allow            443 563;
     proxy_connect_connect_timeout  10s;
     proxy_connect_read_timeout     10s;
     proxy_connect_send_timeout     10s;

     # forward proxy for non-CONNECT request
     location / {
         proxy_pass http://$host;
         proxy_set_header Host $host;
     }
 }
```

Example for curl
----------------

With above configuration, you can get any https website via HTTP CONNECT tunnel.
A simple test with command `curl` is as following:

```
$ curl https://github.com/ -v -x 127.0.0.1:3128
*   Trying 127.0.0.1...                                           -.
* Connected to 127.0.0.1 (127.0.0.1) port 3128 (#0)                | curl creates TCP connection with nginx (with proxy_connect module).
* Establish HTTP proxy tunnel to github.com:443                   -'
> CONNECT github.com:443 HTTP/1.1                                 -.
> Host: github.com:443                                         (1) | curl sends CONNECT request to create tunnel.
> User-Agent: curl/7.43.0                                          |
> Proxy-Connection: Keep-Alive                                    -'
>
< HTTP/1.0 200 Connection Established                             .- nginx replies 200 that tunnel is established.
< Proxy-agent: nginx                                           (2)|  (The client is now being proxied to the remote host. Any data sent
<                                                                 '-  to nginx is now forwarded, unmodified, to the remote host)

* Proxy replied OK to CONNECT request
* TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256  -.
* Server certificate: github.com                                   |
* Server certificate: DigiCert SHA2 Extended Validation Server CA  | curl sends "https://github.com" request via tunnel,
* Server certificate: DigiCert High Assurance EV Root CA           | proxy_connect module will proxy data to remote host (github.com).
> GET / HTTP/1.1                                                   |
> Host: github.com                                             (3) |
> User-Agent: curl/7.43.0                                          |
> Accept: */*                                                     -'
>
< HTTP/1.1 200 OK                                                 .-
< Date: Fri, 11 Aug 2017 04:13:57 GMT                             |
< Content-Type: text/html; charset=utf-8                          |  Any data received from remote host will be sent to client
< Transfer-Encoding: chunked                                      |  by proxy_connect module.
< Server: GitHub.com                                           (4)|
< Status: 200 OK                                                  |
< Cache-Control: no-cache                                         |
< Vary: X-PJAX                                                    |
...                                                               |
... <other response headers & response body> ...                  |
...                                                               '-
```

The sequence diagram of above example is as following:

```
  curl                     nginx (proxy_connect)            github.com
    |                             |                          |
(1) |-- CONNECT github.com:443 -->|                          |
    |                             |                          |
    |                             |----[ TCP connection ]--->|
    |                             |                          |
(2) |<- HTTP/1.1 200           ---|                          |
    |   Connection Established    |                          |
    |                             |                          |
    |                                                        |
    ========= CONNECT tunnel has been established. ===========
    |                                                        |
    |                             |                          |
    |                             |                          |
    |   [ SSL stream       ]      |                          |
(3) |---[ GET / HTTP/1.1   ]----->|   [ SSL stream       ]   |
    |   [ Host: github.com ]      |---[ GET / HTTP/1.1   ]-->.
    |                             |   [ Host: github.com ]   |
    |                             |                          |
    |                             |                          |
    |                             |                          |
    |                             |   [ SSL stream       ]   |
    |   [ SSL stream       ]      |<--[ HTTP/1.1 200 OK  ]---'
(4) |<--[ HTTP/1.1 200 OK  ]------|   [ < html page >    ]   |
    |   [ < html page >    ]      |                          |
    |                             |                          |
```

Example for browser
-------------------

You can configure your browser to use this nginx as PROXY server.

* Google Chrome HTTPS PROXY SETTING: [guide & config](https://github.com/chobits/ngx_http_proxy_connect_module/issues/22#issuecomment-346941271) for how to configure this module working under SSL layer.


Example for Basic Authentication
--------------------------------

We can do access control on CONNECT request using nginx auth basic module.  
See [this guide](https://github.com/chobits/ngx_http_proxy_connect_module/issues/42#issuecomment-502985437) for more details.


Install
=======

Select patch
------------

* Select right patch for building:

| nginx version | enable REWRITE phase | patch |
| --: | --: | --: |
| 1.4.x ~ 1.12.x   | NO  | [proxy_connect.patch](patch/proxy_connect.patch) |
| 1.4.x ~ 1.12.x   | YES | [proxy_connect_rewrite.patch](patch/proxy_connect_rewrite.patch) |
| 1.13.x ~ 1.14.x  | NO  | [proxy_connect_1014.patch](patch/proxy_connect_1014.patch) |
| 1.13.x ~ 1.14.x  | YES | [proxy_connect_rewrite_1014.patch](patch/proxy_connect_rewrite_1014.patch) |
| 1.15.2           | YES | [proxy_connect_rewrite_1015.patch](patch/proxy_connect_rewrite_1015.patch) |
| 1.15.4 ~ 1.16.x  | YES | [proxy_connect_rewrite_101504.patch](patch/proxy_connect_rewrite_101504.patch) |
| 1.17.x ~ 1.18.0  | YES | [proxy_connect_rewrite_1018.patch](patch/proxy_connect_rewrite_1018.patch) |
| 1.19.x           | YES | [proxy_connect_rewrite_1018.patch](patch/proxy_connect_rewrite_1018.patch) |

| OpenResty version | enable REWRITE phase | patch |
| --: | --: | --: |
| 1.13.6 | NO  | [proxy_connect_1014.patch](patch/proxy_connect_1014.patch) |
| 1.13.6 | YES | [proxy_connect_rewrite_1014.patch](patch/proxy_connect_rewrite_1014.patch) |
| 1.15.8 | YES | [proxy_connect_rewrite_101504.patch](patch/proxy_connect_rewrite_101504.patch) |
| 1.17.8 | YES | [proxy_connect_rewrite_1018.patch](patch/proxy_connect_rewrite_1018.patch) |
| 1.19.3 | YES | [proxy_connect_rewrite_1018.patch](patch/proxy_connect_rewrite_1018.patch) |

* `proxy_connect_<VERSION>.patch` disables nginx REWRITE phase for CONNECT request by default, which means `if`, `set`, `rewrite_by_lua` and other REWRITE phase directives cannot be used.
* `proxy_connect_rewrite_<VERSION>.patch` enables these REWRITE phase directives.

Build nginx
-----------

* Build nginx with this module from source:

```bash
$ wget 
$ wget http://nginx.org/download/nginx-1.18.0.tar.gz
$ tar -xzvf nginx-1.18.0.tar.gz
$ cd nginx-1.18.0/
$ yum install openssl-devel pcre-devel zlib-devel gcc gcc+c++ make pcre-devel  zlib-devel patch 
$ patch -p1 < ../ngx_http_proxy_connect_module/patch/proxy_connect.patch
$ ./configure --add-module=/path/to/ngx_http_proxy_connect_module
$ make && make install
```

Build as a dynamic module
-------------------------

* Starting from nginx 1.9.11, you can also compile this module as a dynamic module, by using the `--add-dynamic-module=PATH` option instead of `--add-module=PATH` on the `./configure` command line.

```bash
$ $ wget http://nginx.org/download/nginx-1.18.0.tar.gz https://github.com/yaoshuyin/ngx_http_proxy_connect_module/archive/v0.0.1.tar.gz
$ tar -xzvf nginx-1.18.0.tar.gz
$ tar -xvf v0.0.1.tar.gz
$ cd nginx-1.18.0/
$ patch -p1 < ../ngx_http_proxy_connect_module-0.0.1/patch/proxy_connect.patch
$ ./configure --prefix=/opt/nginx_proxy --with-http_ssl_module --add-module=../ngx_http_proxy_connect_module-0.0.1/
$ make && make install
```

* And then you can explicitly load the module in your nginx.conf via the `load_module` directive, for example,

```
load_module /path/to/modules/ngx_http_proxy_connect_module.so;
```

* :exclamation: Note that the ngx_http_proxy_connect_module.so file MUST be loaded by nginx binary that is compiled with the .so file at the same time.


Build OpenResty
---------------

* Build OpenResty with this module from source:

```bash
$ wget https://openresty.org/download/openresty-1.19.3.1.tar.gz
$ tar -zxvf openresty-1.19.3.1.tar.gz
$ cd openresty-1.19.3.1
$ ./configure --add-module=/path/to/ngx_http_proxy_connect_module
$ patch -d build/nginx-1.15.8/ -p 1 < /path/to/ngx_http_proxy_connect_module/patch/proxy_connect_rewrite_101504.patch
$ make && make install
```

Test Suite
==========

* To run the whole test suite:

```bash
$ hg clone http://hg.nginx.org/nginx-tests/
$ export TEST_NGINX_BINARY=/path/to/nginx/binary
$ prove -v -I /path/to/nginx-tests/lib /path/to/ngx_http_proxy_connect_module/t/
```

Error Log
=========

This module logs its own error message beginning with `"proxy_connect:"` string.  
Some typical error logs are shown as following:

* The proxy_connect module tries to establish tunnel connection with backend server, but the TCP connection timeout occurs.

```
2019/08/07 17:27:20 [error] 19257#0: *1 proxy_connect: upstream connect timed out (peer:216.58.200.4:443) while connecting to upstream, client: 127.0.0.1, server: , request: "CONNECT www.google.com:443 HTTP/1.1", host: "www.google.com:443"
```

Directive
=========

proxy_connect
-------------

Syntax: **proxy_connect**  
Default: `none`  
Context: `server`  

Enable "CONNECT" HTTP method support.

proxy_connect_allow
-------------------

Syntax: **proxy_connect_allow `all | [port ...] | [port-range ...]`**  
Default: `443 563`  
Context: `server`  

This directive specifies a list of port numbers or ranges to which the proxy CONNECT method may connect.  
By default, only the default https port (443) and the default snews port (563) are enabled.  
Using this directive will override this default and allow connections to the listed ports only.

The value `all` will allow all ports to proxy.

The value `port` will allow specified port to proxy.

The value `port-range` will allow specified range of port to proxy, for example:

```
proxy_connect_allow 1000-2000 3000-4000; # allow range of port from 1000 to 2000, from 3000 to 4000.
```

proxy_connect_connect_timeout
-----------------------------

Syntax: **proxy_connect_connect_timeout `time`**  
Default: `none`  
Context: `server`  

Defines a timeout for establishing a connection with a proxied server.


proxy_connect_read_timeout
--------------------------

Syntax: **proxy_connect_read_timeout `time`**  
Default: `60s`  
Context: `server`  

Defines a timeout for reading a response from the proxied server.  
The timeout is set only between two successive read operations, not for the transmission of the whole response.  
If the proxied server does not transmit anything within this time, the connection is closed.

proxy_connect_send_timeout
--------------------------

Syntax: **proxy_connect_
