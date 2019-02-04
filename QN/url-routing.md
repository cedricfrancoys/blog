
## What is a route ?
A route associates a URI with a resolver which indicates how to handle the request and might be as many things as a class name, a function, a script path or even another URI.


## Syntax
```
{
"URI" : {
	"METHOD" : {
		["description": "",]
		["operation": "",]
                ["redirect": ""]
	}
	[,
	"METHOD" : {
		["description": "",]
		"operation": ""
	}[,...]	
	]
[,...]
}
```

alternate notation
``` 
{
"URI" : "redirection"
[,...]
}
```

As an arbitrary convention, we can limit the fields to "description" and "operation":

* The description only serves to allow developer(s) to remember routes roles in the config file
* The operation attribute is either a query string that can be resolved to a script path (controller) or another URI the route actually points to.

To distinguish an operation from a redirection, we can use the following convention: 
an operation is a call to the main entry point (and should therfore always contain either '?get', '?do', or '?show'), whereas a redirection is an absolute URL (starting with a '/') to a route defined elsewhere.
	


## Parameters
ability to map some parameters using 

A common route notation is to use the :[param] syntax with '?' (wildcards notation) as optional flag
```
    "/search/:q?":"/resipedia.fr#!/search/:q"
```

## Translation

In most context, URI rewriting goal is to name operations in a logicial and intuitive way but also to ease URI reading for user and .
Therefore, it is relevant to consider the user language.

We can use additional config files (.json) and use the alternation notation (redirection) to map language-specific routes to the previously defined ones.

## HTTPD URI handling

The easiest way to implement this is to define a rewrite rule on the HTTP server along with an entry point having routing capabilities:
.htaccess
```
    <IfModule mod_rewrite.c>

        RewriteEngine On
        RewriteBase /
        RewriteRule ^index\.php(\??.*)$ - [L]

        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule . /index.php [L]

    </IfModule>
```

which in turn can be in_array('mod_rewrite', apache_get_modules());
if(!function_exists('apache_get_modules') )

if( strpos( $_SERVER['SERVER_SOFTWARE'], 'Apache') !== false) 

$_SERVER['SERVER_SOFTWARE']
apache_get_version();

makes it easy to read and to detect inconsistencies or non-logic routes
```
{
    "/api/categories": {
        "GET": {
               "description: "provide categories listing",
               "operation": "?get=myapp_category_list&api=1.0"
        },
        "DELETE": {
               "description: "categories bulk deletion",
               "operation": "?do=myapp_category_delete"
        }

}
```


Thoses routes can be easily mapped to operations as described in the article [Trivial scripts as controllers](dev/articles/trivial-script-as-controller/)


## Enrich response description

Additional informations might be added, describing the response that the operation will return: 

* What kind of data is returned (content-type/MIME) : `application/json`, application/vnd.api+json`, `text/xml`, `application/pdf`, ...
* What charset is used for text-encoding: `UTF-8`, `ISO-8859-1`, ...
* What is the scope of the controller : does it allow cross-origin requests / from which hosts


Here is an example on how to use the announce method to set those descriptors: 
```
QNLib::announce([
    'response'      => [
        'content-type'  => 'text/xml',
        'charset'       => 'utf-8'
        'allow-origin'  => '*'
    ]
]);
```


Finally, here is how the existing routes can be retrieved and returned as JSON with their full description:

```php
    use config\QNLib;

    list($params, $providers) = QNLib::announce([
        'description'   => 'List of existing routes',
        'providers'     => ['context', 'route'] 
        'response'      => [
            'content-type'  => 'application/json'
        ]
    ]);

    list($context, $router) = [$providers['context'], $providers['route']];

    $routes = $router->add(QN_BASE_DIR.'/config/routing/*.json')->getRoutes();

    $result = [];

    foreach($routes as $path => $resolver) {
        $batch = $router->normalize($path, $resolver);
        foreach($batch as $route) {
            if(isset($route['redirect']) || $route['operation']['type'] == 'show') continue;

            $route['operation']['params']['announce'] = true;

            $json = QNLib::run($route['operation']['type'], 
                               $route['operation']['name'], 
                               $route['operation']['params']);
            
            $result[] = json_decode($json, true);
        }
    }

    $context->httpResponse()->body($result)->send();
``` 

## Read more
Here are some links about how a few popular PHP frameworks deal with routes :
* https://www.codeigniter.com/userguide3/general/routing.html
* https://www.slimframework.com/docs/objects/router.html
