## Trivial scripts as controllers

This article unveils an original approach for building API in a way that eases code re-use, encourages self-documented code and allows operations to be used in various contexts (CLI, HTTP).

![](//i.imgur.com/wUwqlDWl.jpg)

## Operations

Should we be in a RESTful context or not, a request made to a server is always a solicitation to perform an operation and can be interpreted as one of those 3 verbs:

1.  **'do'** something (Action handler)
2.  **'get'** something (Data provider)
3.  **'show'** something (App provider)

So, basically we only have three types of operations which, in turn, can easily be mapped with the methods defined by the HTTP protocol:

*   POST : create (action handler)
*   GET : read (data or app provider)
*   PUT : replace (action handler)
*   PATCH : update (action handler)
*   DELETE : delete (action handler)

## Routing

The underlying idea suggested by this article is that it is possible to decompose an application logic into trivial controllers.

To achieve so, all it takes is associate each route with an 'operation' and to define a single entry-point that routes the request to the appropriate controller through a 'run' method.

In a web server context, if we use index.php as entry-point, the query string of the URI should bear the following info:
```
    ?request_type=[public|private:]path_to_controller
```

along with some optional parameters.

The advantage of such architecture is that it allows to be invoked in similar ways whatever the context:

*   server-side
    *   using CLI: `php run.php --get=public:qinoa_tests --id=1 --test=2 --announce=true`
    *   inside the code: `run('get', 'public:qinoa_tests', ['id'=>1, 'test'=>2]);`
*   client-side through HTTP: `/index.php?get=qinoa_tests&id=1&test=2`

For instance a controller can easily rely on the result of another controller, being invoked inside a same request.

## Scripts as controllers

In that logic, a controller is nothing more than a PHP script in charge of:

*   requesting the services its requires (dependencies);
*   defining the parameters it expects, along with related types and constraints;
*   specifying the format it uses for the response.

In addition, it would be nice for every controller to provide a short description of what it does and what is its intended usage.

Here is an example of such an `announce()` method from file `public/packages/demo/data/books/suggestions.php`
```php
    list($params, $providers) = QNLib::announce([
        'description'   => 'Suggests bedtime stories based on a child age and interest',
        'params'        => [
            'keywords' => [
                'description'   => "Keywords that catch your child attention",
                'type'          => 'array',
                'default'       => []
            ],
            'age' => [
                'description'   => 'Your child age',
                'type'          => 'integer',
                'min'           => 4,
                'required'      => true
            ]
        ],
        'response'      => [
            'content-type'  => 'application/json',
            'charset'       => 'utf-8'
        ],
        'providers'     => ['orm', 'context'] 
    ]);
```

And here the sample code that handles the valid requests:
```php
    list($context) = [$providers['context']];
    
    $store = [
      4  => ['Goldielocks and the three bears', 'Three little pigs'],
      5  => ["Charlotte's web", 'The Little Prince'],
      8  => ['Charly and the chocolate factory', 'Alice in Wonderland'],
      10 => ['Harry Potter', 'The Jungle book']
    ];
    
    $result = [];
    
    foreach($store as $age => $books) {
        if($age <= $params['age']) {
            if(count($params['keywords'])) {
                foreach($params['keywords'] as $keyword) {
                    foreach($books as $book) {
                        if(stripos($book, $keyword) !== false) {
                            $result[] = $book;
                        }
                    }
                }
            }
            else {
                $result = $result + $books;
            }
        }
        else break;
    }
    
    $context->httpResponse()->body($result)->send();
```

This controller can then be invoked these ways:

*   CLI: `php run.php --get=demo_books_suggestions --keywords=bears,chocolate --age=10`
*   HTTP: `/index.php?get=demo_books_suggestions&age=4`

Finally, here is an excerpt of (a simplified version of) the `run()` method:
```php
    public static function run($type, $operation, $body=[], $root=false) {
      $result = '';
      $resolved = [
          'type'      => $type,       // 'do', 'get' or 'show'
          'operation' => null,        // {package}_{script_path}
          'visibility'=> 'public',    // 'public' or 'private'
          'package'   => null,        // {package}   
          'script'    => null         // {path/to/script.php}
      ];
      // define valid operations specifications
      $operations = array(
          'do'    => array('kind' => 'ACTION_HANDLER','dir' => 'actions'),    
          'get'   => array('kind' => 'DATA_PROVIDER','dir' => 'data'), 
          'show'  => array('kind' => 'APPLICATION','dir' => 'apps')  
      );
      // retrieve services container instance   
      $container = Container::getInstance();    
      $context = $container->get('context');
      // adapt current request
      $request = $context->httpRequest();
      $request->body($body);
      // extract parts from given operation
      $operation = explode(':', $operation);
      if(count($operation) > 1) {
          $visibility = array_shift($operation);
          if($visibility == 'private') $resolved['visibility'] = $visibility;
      }
      $resolved['operation'] = $operation[0];
      // include resolved script, if any
      if(isset($operations[$resolved['type']])) {
          $operation_conf = $operations[$resolved['type']];
          // store current operation into context
          $context->set('operation', $resolved['operation']);
          $filename = 'packages/'.
                      $resolved['package'].'/'.
                      $operation_conf['dir'].'/'.
                      $resolved['script'];
          // set current dir according to visibility (i.e. 'public' or 'private')
          chdir(QN_BASE_DIR.'/'.$resolved['visibility']);
          include($filename); 
      }
    }
```