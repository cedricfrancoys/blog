# Overcoming CORS when a data provider does not



Despite the fact that data exchange through API has become a common thing, many data providers still focus on back-end interactions (i.e. server to server communications) while more and more applications might interact directly through the final environment (which, in a web context, is often the customer web browser).

![](//i.imgur.com/rIraAL2.jpg)

In the meanwhile web navigation security has been increasing, mostly to prevent the final user from accessing unwanted and potentially harmful content and or script behavior.

More specifically, during the last decade a new mechanism, known as CORS (for Cross-Origin Resource Sharing), has been added to all major web browsers. 

It prevents scripts run from a website to request data from a provider which does not explicitly allow that website to do so.


In most cases there is no vulnerability involved and allowing any HTTP request is as simple as adding one line in the header: 

```
Access-Control-Allow-Origin: *
```



However, for many reasons, it is sometimes just not possible to make the provider do that update.

In such cases, setting up a proxy is the only option. Besides it might be a good practice, in case there are several tools using the targeted external API, to define a central service in the environment as sole internal node for accessing external data.

The role of that proxy is just to mimic the request the client need the response to, send it to the data provider, intercept the response and append a `Access-Control-Allow-Origin: *` to it before sending the result back to the client.

In another article, I also explain why it is wise and how it can reduce complexity when numerous external data providers are involved.



Finally, for those familiar with cURL, here is a short and non-exhaustive snippet of such CORS proxy :

```
$url = 'https://'.SERVICE_PROVIDER_FQDN.$_SERVER['REQUEST_URI'];

// build the cURL request
$ch = curl_init();
curl_setopt($ch, CURLOPT_HTTPHEADER, getallheaders());
curl_setopt($ch, CURLOPT_URL, $url);
curl_setopt($ch, CURLOPT_HEADER, true);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
// handle POST requests
if($_SERVER['REQUEST_METHOD'] == 'POST') {
        curl_setopt($ch, CURLOPT_POST, 1);
        curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($_POST));
}
// send cURL request
$result = curl_exec($ch);
curl_close( $ch );

// relay the received header
list($headers_raw, $body) = explode("\r\n\r\n", $result);
$headers = [];
$lines = explode("\n", $headers_raw);
foreach($lines as $header) {
    list($key, $value) = explode(':', $line); 
    $key = trim($key);
    if(!isset($headers[$key])) {
	    header($header);
	    $headers[$key] = $value;            
    }
}

// append CORS allow all to the header
header('Access-Control-Allow-Origin: *');  
// and output the body
print($body);
```