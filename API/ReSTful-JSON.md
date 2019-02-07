application/json

JSONRequest
application/jsonrequest


JSEND
https://labs.omniti.com/labs/jsend
https://technick.net/guides/software/software_json_api_format/

JSON API
http://jsonapi.org/
application/vnd.api+json

JSON-LD
https://fr.wikipedia.org/wiki/JSON-LD

OData (Microsoft)
"application/json;odata=verbose"

HATEOAS
https://en.wikipedia.org/wiki/HATEOAS

GRAPHQL
https://graphql.org/



Other languages used for API description and design
RAML
https://raml.org/

Open API
https://swagger.io/specification/


Status is in HTTP header

if it's not 200, it is probably an error

in case an error occured, response should contain some details about it


if error is user-reated (4XX) that message should allow him to fix his request 


client-side : JS/TS

XMLHttpRequest (https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest)

	status		(unsigned short)	HTTP status code
	statusText 	(string)		HTTP status message ietf rfc 2616 section 6.1.1
	responseType				'text', 'json', document', 'blob', 'arraybuffer'
	response	DOMString, JavaScript object, Document, Blob, ArrayBuffer



### AngularJS

$http.post({
	url : my_url,
	data : txpostdata,
	headers : { 'Content-Type' : 'application/json' },
}).then(
function (response) {
	var payload = response.data;
});

### Angular

this.http
.request(my_url)
.subscribe(response => payload = response.json());

### Native Javascript

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