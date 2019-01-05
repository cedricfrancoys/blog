# Enrich PHP for HTTP messages


This article aims to explore a way to enrich the native PHP capabilities in order to handle HTTP messages from and to RESTful APIs without the need of external dependencies.

![](http://i.imgur.com/k81Snodl.jpg)

## Intro

Until a few years ago the majority of HTTP requests were still GET and POST messages. For that reason, since its early development, PHP has always focused on providing data about the request being handled by using global vars (`$_SERVER`, `$_GET`, `$_POST`, ...) and only natively recognize the `application/x-www-form-urlencoded` content-type along with URL-encoded data from the query string (e.g. GET and POST methods)

This article unveils a proof-of-concept allowing to:

*   Handle all **HTTP methods** and **content types**
*   Take advantage of the **native stream functions** without having to deal with low-level PHP libraries nor manually set HTTP context and options
*   Harmonize the way to handle **concepts** that are **logically identical** (HTTP messages can be sent or received by the current script)

While focusing on :

*   clear code
*   understandability
*   integration and maintenance ease

### What is this about ? - Code glimpse

#### This article shows how to design a few classes allowing to...
```php
    // 1) Send a HTTP request to external services
        
    $request = new HttpRequest('GET https://graph.facebook.com/v2.9/me HTTP/1.1');
    /* Also valid:
    $request = new HttpRequest('https://graph.facebook.com/v2.9/me');
    $request = new HttpRequest('/v2.9/me', ['Host' => 'graph.facebook.com:443']);
    */
    
    // send a HTTP request to an external API service
    $response = $request
    ->body([
                'fields'       => 'email,first_name,last_name',
                'access_token' => $my_access_token
              ])
    ->send();
    
    // 2) and send a HTTP response back to the client
    
    // output JSON response with retrieved data
    $context->httpResponse()
    ->header('Content-Type', $response->header('Content-Type'))
    ->body([
               'result' => [ 
                   'email' => $response->get('last_name')
               ]
    ])
    ->send();
```

### Why would anyone need this ?

Nowadays, REST APIs are getting more and more common. Therefore ability to send and receive HTTP messages is a crucial point for a web server.

PHP has everything it takes to offer such ability. So let's see how to provide HTTP utilities in an efficient, easy to maintain and yet elegant manner.

Let's start with the very beginning :

### What is a HTTP request ?

Along with HTTP responses, HTTP requests are the forms that the HTTP protocol uses for exchanging messages between a client and a server.

A HTTP message is actually very simple and has the following structure:
```
     Headline (request-line for request and status-line for response)
     Header (CRLF separated lines)
     [empty line]
     Body
```

HTTP requests are messages for asking something to the server, e.g. "I would like to GET that resource".

Here is a typical such request:
```
        GET /page.html HTTP/1.0
        Host: example.com
        Accept: text/html
        Accept-Encoding: *
        User-Agent: Mozilla/4.0 (compatible; MSIE5.01; Windows NT)
```

Where

*   `example.com` is the server the request is sent to
*   `GET` is the kind of action the client ask the server to perform ("method")
*   `HTTP/1.0` is the protocol used
*   `/page.html` is the server path to the requested resource

### This is not a resource

Even if terms "URI" and "resource" can often be used interchangeably, it is good to remember that a URI (Uniform Resource Identifier) is not the resource itself but a description of how it can be retrieved.

Generic form:
```
    scheme:[//[user[:password]@]host[:port]][/path][?query][#fragment]
```

Examples:

*   [http://www.example.com/index.html](http://www.example.com/index.html)
*   [https://www.w3.org/hypertext/DataSources/Overview.html](https://www.w3.org/hypertext/DataSources/Overview.html)
*   [ftp://me:mypass@ftp.example.com:80/index.html](ftp://me:mypass@ftp.example.com:80/index.html)

The [wikipedia article about URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) offers an excellent visual summary :

                          hierarchical part
            ┌────────────────────┴────────────────────┐
                          authority             path
            ┌────────────────┴──────────────┐┌────┴───┐
      abc://username:password@example.com:123/path/data?key=value&key2=value2#fragid
      └┬┘   └──────┬────────┘ └────┬────┘ └┬┘           └─────────┬─────────┘ └─┬──┘
    scheme  user information      host    port                  query        fragment


### PHP context

Natively HTTP requests are pre-processed: most headers are stored in the `$_SERVER` super global variable and content can be retrieved in `$_REQUEST`, `$_FILES`, `$_COOKIE`.

In times of OAuth handshakes and RESTful architecture, we need a little more capabilities to be able to take advantage of the full HTTP protocol specifications and possibilities.

But not that much... Here is all we need to fully represent HTTP messages :

*   Method (GET and POST, but also HEAD, PUT, PATCH, DELETE, PURGE, OPTIONS, TRACE, CONNECT)
*   URI
*   Headers
*   Body
*   Protocol (e.g. HTTP/1.1)
*   Status

Let's see how to retrieve all these values...

#### URI

To build the currently requested URI we need to retrieve :

*   scheme
*   user information (username and password, if any)
*   host
*   port (might be guessed, based on scheme)
*   path
*   query (not necessarily present)
*   fragment (most browsers don't relay that part to the server)

Path and query can be retrieved in the `$_SERVER['REQUEST_URI']` global var.  
From there, it is not too difficult to retrieve the full URI:
```php
        function getHttpUri() {
            $scheme = isset($_SERVER['HTTPS']) ? "https" : "http";
            $auth = '';
            if(isset($_SERVER['PHP_AUTH_USER']) && strlen($_SERVER['PHP_AUTH_USER']) > 0) {
                $auth = $_SERVER['PHP_AUTH_USER'];
                if(isset($_SERVER['PHP_AUTH_PW']) && strlen($_SERVER['PHP_AUTH_PW']) > 0) {
                    $auth .= ':'.$_SERVER['PHP_AUTH_PW'];
                }
                $auth .= '@';
            }
            $host = isset($_SERVER['HTTP_HOST'])?$_SERVER['HTTP_HOST']:'localhost';
            $port = isset($_SERVER['SERVER_PORT'])?$_SERVER['SERVER_PORT']:80;
            return  $scheme."://".$auth."$host:$port{$_SERVER['REQUEST_URI']}";
        }
```

#### Method

Some user agent or firewall might prevent PUT or DELETE methods from being sent directly. In that case a POST method is sent and the X-HTTP-Method-Override header is used to override to intended method.
```php
        function getHttpMethod() {
            static $method = null;        
            if(!$method) {
                $method = $_SERVER['REQUEST_METHOD'];
                // in case of POST, check X-HTTP-Method-Override header
                if (strcasecmp($method, 'POST') === 0) {
                    if (isset($_SERVER['X-HTTP-METHOD-OVERRIDE'])) {
                        $method = $_SERVER['X-HTTP-METHOD-OVERRIDE'];
                    } 
                }
                // normalize to upper case
                $method = strtoupper($method);
            }
            return $method;        
        }
```

#### Headers

PHP provide (when running with Apache) a convenient function to retrieve HTTP headers from request being handled: `getallheaders()`.

What we need to do is:

*   provide a polifill function in case `getallheaders` is not available (PHP running with non-apache web server: nginx, lighttpd, ...)
*   normalize headers to handle non-standard but commonly used headers such as `Authorization`, `ETag` and `X-Forwarded-For`

#### Content

PHP only natively handle the `application/x-www-form-urlencoded` content-type along with URL-encoded data from the query string and stores those, respectively, in the `$_POST` and `$_GET` super globals.  
Let's see how to:

1. Fetch request body whatever the content-type (using `php://input` stream)
2. Retrieve pre-processed data from GET and POST requests (using `$_REQUEST` super global)
```php
       function getHttpBody() {
           $body = '';        
           // retrieve current method
           $method = $this->getHttpMethod();        
           // append parameters from request URI if not already in (e.g. internal redirect)
           if($method == 'GET') {            
               if(false !== strpos($_SERVER['REQUEST_URI'], '?')) {
                   $params = [];            
                   parse_str(explode('?', $_SERVER['REQUEST_URI'])[1], $params);  
                   $_REQUEST = array_merge($_REQUEST, $params);            
               }                    
           }        
           // use PHP native HTTP request parser for supported methods 
           if( in_array($method, ['GET', 'POST']) && !empty($_REQUEST) ){            
               $body = $_REQUEST;
           }
           // otherwise load raw content from input stream 
           else {            
               $body = @file_get_contents('php://input');            
           }        
           return $body;
       }
```

We now have a body that is either a string or an array. An additional method can be defined to try to normalize the body into an associative array, based on `Content-Type` header, and fallback to a raw string if conversion is not possible (see an example of such method below).

### Data structure modeling

Now comes the tricky part : « How to turn all this into a convenient model ? »

#### HTTP message

As mentioned ealier, HTTP requests and responses are quite similar and can be modelized from a common structure, **HttpMessage**, which constructor has only 3 arguments:

*   headline
*   headers
*   body

From there, we are able to retrieve all we need to build a HTTP message. All messages have parts in common:

*   method : first word in request headline, last word in response headline (`GET` method can be used as default)
*   protocol : last word in request headline, first word in response headline (`HTTP/1.1` can be used as default)
*   status : second and third words from requests headline (this might be used for requests as well, e.g. to determine if a request has already been sent)
*   body
*   URI
*   headers (which consist in a series of `header_name`: `header_value` tuples)

**HttpRequest** and **HttpResponse** classes (see below) inherit from HttpMessage class which holds one **HttpUri** and one **HttpHeaders** as private members (see below).

#### Body

When possible, the content is turned into an associative array, based on the Content-Type header. This can easily be done at least for the most common MIME types : `application/x-www-form-urlencoded`, `application/json`, `text/javascript`, `text/xml`. Here is just a short example that could be improved to handle more content-types:
```php
        function setBody($body) {
            if(!is_array($body)) {
                switch($this->getHeaders()->getContentType()) {
                case 'application/x-www-form-urlencoded':
                    $params = [];
                    parse_str($body, $params);
                    $body = (array) $params;
                    break;
                case 'application/json':
                case 'text/javascript':
                    $body = json_decode($body, true);
                    break;
                case 'text/xml':
                case 'application/xml':
                case 'text/xml, application/xml':
                    $xml = simplexml_load_string($body, "SimpleXMLElement", LIBXML_NOCDATA);
                    $json = json_encode($xml);
                    $body = json_decode($json, true);                
                }
            }        
            $this->body = $body;
            return $this;
        }
```

#### URI

To handle URI parts, we'll use a **HttpUri** class. This nomenclature might sound a little redundant but this name actually makes sense in order to have uniform class names and to insist on the fact that not all URI apply to a web context.

For instance, "urn:isbn:0-395-36341-1" is a valid URI but useless for HTTP protocol.

PHP provide two very convenient function: `parse_url($uri)` and `filter_var($uri, FILTER_VALIDATE_URL)`.

We just need a tiny additional work for providing support for internationalized domain name (IDN) support.
```php
        public function setUri($uri) {
            if(self::isValid($uri)) {
                $this->parts = parse_url($uri);
            }
            return $this;
        }
    
        public static function isValid($uri) {
            $res = filter_var($uri, FILTER_VALIDATE_URL);
            if (!$res) {
                // check if URI contains unicode chars
                $mb_len = mb_strlen($uri);
                if ($mb_len !== strlen($uri)) {
                    // replace all multi-bytes chars with a always-valid single-byte char (A)
                    $safe_uri = '';
                    for ($i = 0; $i < $mb_len; ++$i) {
                        $ch = mb_substr($uri, $i, 1);
                        $safe_uri .= strlen($ch) > 1 ? 'A' : $ch;
                    }
                    // re-check normalized URI
                    $res = filter_var($safe_uri, FILTER_VALIDATE_URL);
                }
            }
            return $res;
        }
```

We'll aslo provide Setters and Getters :

*   `getScheme`, `getUser`, `getPassword`, `getHost`, `getPort`, `getPath`, `getQuery`, `getFragment`
*   `setScheme`, `setUser`, `setPassword`, `setHost`, `setPort`, `setPath`, `setQuery`, `setFragment`

Plus: setters return the current instance to allow calls chaining.

In addition, we'll take advantage of the \_\_toString magic method to allow casting the object to a string and build the resulting URI.
```php
        public function __toString() {
            $uri = '';
            $user_info = '';
            if(isset($this->parts['user']) && strlen($this->parts['user']) > 0) {
                $user_info = $this->parts['user'];
                if(isset($this->parts['pass']) && strlen($this->parts['pass']) > 0) {
                    $user_info .= ':'.$this->parts['pass'];
                }
                $user_info .= '@';
            }
            $query = $this->getQuery();
            $fragment = $this->getFragment();
            if(strlen($fragment) > 0) {
                $query = $query.'#'.$fragment;
            }
            if(strlen($query) > 0) {
                $query = '?'.$query;
            }
            return $this->getScheme().'://'.
                   $user_info.$this->getHost().':'.$this->getPort().
                   $this->getPath().$query;
        }
```

#### Headers

A HTTP header almost never comes alone. Therefore, we'll define a **HttpHeaders** class that will act as an array of headers. In case of a fresh request, no header is pre-defined.

Here are Setters and Getters :

*   `getHeader`, `getHeaders`, `getCharset`, `getCharsets`, `getLanguage`, `getLanguages`, `getIpAddress`, `getIpAddresses`, `getContentType`
*   `setHeader`, `setHeaders`, `setCharset`, `setCharsets`, `setLanguage`, `setLanguages`, `setIpAddress`, `setIpAddresses`, `setContentType`

Plus: setters return the current instance to allow calls chaining.

#### Request

**HttpRequest** constructor is in charge of retrieving method, URI and protocol. Method should be the first word of the headline but we can easily add some flexibilty.
```php
        public function __construct($headline='', $headers=[], $body='') {
            parent::__construct($headline, $headers, $body);        
            // parse headline
            $parts = explode(' ', $headline, 3);        
            // 1) retrieve protocol
            if(isset($parts[2])) {
                $this->setProtocol($parts[2]);
            }
            // 2) retrieve URI and host
            if(isset($parts[1])) {
                // URI ?
                if(HttpUri::isValid($parts[1])) {
                    $this->setUri($parts[1]);
                }
                else {
                    // check Host header for a port number (see RFC2616)
                    [...]
                }
            }
            // 3) retrieve method
            if(isset($parts[0])) {
                // method ?
                if(in_array($parts[0], self::$HTTP_METHODS) ) {
                    $this->setMethod($parts[0]);
                }
                else {
                    // URI ?
                    if(HttpUri::isValid($parts[0])) {
                        $this->setUri($parts[0]);
                    }
                    else {
                        // check Host header for a port number (see RFC2616)
                        [...]
                        }                
                    }                
                }
            }
    
        }
```

#### Response

**HttpResponse** constructor is in charge of retrieving protocol and status. Protocol should be the first word of the headline but we can easily add some flexibility.
```php
        public function __construct($headline, $headers=[], $body='') {
            parent::__construct($headline, $headers, $body);
    
            // parse headline
            $parts = explode(' ', $headline, 2);
    
            // retrieve status and/or protocol
            if(isset($parts[1])) {
                $this->setStatus($parts[1]);
                $this->setProtocol($parts[0]);
            }
            else {
                if(isset($parts[0])) {
                    if(is_numeric($parts[0])) {
                        $this->setStatusCode($parts[0]);
                    }
                    else {
                        $this->setProtocol($parts[0]);
                    }
                }
            }
    
        }
```

#### Pure OO vs. Helpers

Turning everything into object-models or not depends on the developing environment. Helpers might be easier to use in a procedural style library, on the other hand, objects will obviously be preferred in a pure Object-Oriented framework...

In our case, the most important data structures (URI and HTTP headers) consist of little information that can easily fit into an associative array. So, in addition of Objects, we'll also define some Helper classes that will allow easy re-use in non-OO contexts.

Here are their signatures:
```php
    string HttpUriHelper::getScheme(string $uri);
    string HttpUriHelper::getHost(string $uri);
    string HttpUriHelper::getPort(string $uri);
    string HttpUriHelper::getPath(string $uri);
    string HttpUriHelper::getQuery(string $uri);
    string HttpUriHelper::getFragment(string $uri);    
    string HttpUriHelper::getUser(string $uri);
    string HttpUriHelper::getPassword(string $uri);
    string HttpUriHelper::getBasePath(string $uri);
    
    array HttpHeaderHelper::getCharsets(array $headers);
    array HttpHeaderHelper::getLanguages(array $headers);
    array HttpHeaderHelper::getIpAddresses(array $headers);
    string HttpHeaderHelper::getCharset(array $headers);
    string HttpHeaderHelper::getLanguage(array $headers);
    string HttpHeaderHelper::getIpAddress(array $headers);
    string HttpHeaderHelper::getContentType(array $headers);
```

## Conclusion

We've seen that it is possible to:

*   Normalize **PHP context** through a dedicated `PhpContext` Singleton;
*   Communicate with **external HTTP services** throug `HttpRequest` and `HttpResponse` objects;
*   Automatically interpret responses as native associative arrays **whatever the content-type**.

All this, without the burden of having to rely on a whole framework to benefit of a few additional functionalities.

Of course, this is just a proof of concept and a lot of things could still be improved (for instance to handle `$_FILES`, `$_COOKIE` and `$_SESSION` super globals).

## Download

All sources are freely available in **[this github repository](https://github.com/cedricfrancoys/php-http)**.

## Read more

Links to learn more about next generation PHP:

*   The excellent [ReactPHP project](https://reactphp.org/)
*   IETF [URI structure specs](http://www.ietf.org/rfc/rfc3986.txt)
*   In its [PSR-7](http://www.php-fig.org/psr/psr-7/), the PHP-FIG suggests a similar though more global and pure-OOP approach