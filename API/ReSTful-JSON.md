# ReSTful JSON


For a long time, XML has reigned supreme over applications communications as a universally recognized format for data exchange. Back in the glory days of XUL, it even allowed to communicate with browsers, but for a time only.

![](https://i.imgur.com/irOcoFsl.jpg)

As the future is trending towards the web, the ever-increasing possibilities of ECMA script (aka JavasScript), have favoured the emergence of a new format: JSON (Javascript Object Notation), which offers the advantage to be natively readable by any JS script (without the need of additional transformation).


Even if in a ReSTful context, the exchange format is not subject to any specification, as long as the programming language of the major platform will be ECMA compatible (browsers, webservers and even OS - through transpiling - can use JS/TS), it is a safe guess that JSON will remain a "de-facto" standard for a long while.



### Less overhead, but less precision

The weakness of this format is also its strength : JSON comes with no type definition and only supports basic types (strings, numeric values, sets, arrays, maps). But its structure representation is just versatile enough to allow to represent almost any kind of data.



### JSON is not ReST


Just using JSON packet between client and server does not necessarily means ReST-enabled (Representational state transfer). In fact, developer teams are left on their own to build something that either follows official specifications, or sticks to a custom logic defined during the design phase.

For instance, GraphQL, a widely spread solution promoted by Facebook, uses JSON as query language to describe and request specific piece of data.

Here below is a quick summary of some of the main JSON specifications and their related MIME / Content-Type values :



#### JSON


application/json

#### JSONRequest
"application/jsonrequest"


#### JSEND
https://labs.omniti.com/labs/jsend
https://technick.net/guides/software/software_json_api_format/

####  JSON API
http://jsonapi.org/
"application/vnd.api+json"

#### JSON-LD
https://fr.wikipedia.org/wiki/JSON-LD

#### OData (Microsoft)
"application/json;odata=verbose"

#### HATEOAS
https://en.wikipedia.org/wiki/HATEOAS

#### GraphQL

"application/json" 
alt: "application/graphql" 

https://graphql.org/



### Other languages used for API description and design

#### RAML
https://raml.org/

#### Open API
https://swagger.io/specification/


Status is in HTTP header

if it's not 200, it is probably an error

in case an error occurred, response should contain some details about it

if error is user-related (4XX) that message should allow him to fix his request 


client-side : JS/TS

XMLHttpRequest (https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)

```
status		 (unsigned short) HTTP status code
statusText 	 (string) HTTP status message ietf rfc 2616 section 6.1.1
responseType 'text', 'json', document', 'blob', 'arraybuffer'
response	 DOMString, JavaScript object, Document, Blob, ArrayBuffer
```



## API call examples

Because an article always look more serious with a bit of code, here are some quick snippets to make a typical API call using AJAX :

### AngularJS

```javascript
$http.post({
	url : my_url,
	data : txpostdata,
	headers : { 'Content-Type' : 'application/json' },
}).then(
function (response) {
	var payload = response.data;
});
```

### Angular.io
```javascript
this.http
.request(my_url)
.subscribe(response => payload = response.json());
```

### Native Javascript
```javascript
var req = new XMLHttpRequest();
req.onreadystatechange = function(event) {
    if (this.readyState === XMLHttpRequest.DONE) {
        if (this.status === 200) {
            payload = JSON.parse(this.responseText);
        } 
    }
};
req.open('GET', my_url, true);
req.send();
```