# Errors handling & reporting

This article explores how to harmonize internal logic and generated outputs in a HTTP context (typically a RESTful API) when it comes to handling errors.

![](//i.imgur.com/yxbvYDEl.jpg)

## What are "errors" and "reporting" ?

### Kinds of errors

Basically, there are three kinds of errors:

1.  coding error: mistyped or inconsistent code, raising a compilation or parsing error  
    (under production environment, this should never occur)

2.  logic error: the code does not act according to the underlying logic  
    (most of the time, no error nor warning is raised but unexpected result is returned)

3.  use case error  
    \- algorithm do not handle certain situations that lead to code misbehaviour;  
    \- user provided invalid parameters (wrong configuration; mistyped, misformatted, undefined, or unexpected value).


### Error reporting

Usually, reporting consists of logging or displaying messages that belong to one of those five families:

### Verbose or debug

Extra information for tracing and debugging purpose.

### Notice

Information about something that could indicate an error (or could turn into an error in the future), but could also happen in the normal course of the script execution.

*   Execution is not halted.

**Examples**:

*   Deprecated function usage detected
*   About-to-be deprecated syntax

### Warning

Alert about unexpected but not harming situation.

*   The code encountered a forbidden scenario or something that might lead to unexpected result and prevented/fixed it.
*   Execution is not halted.

**Examples**:

*   a non-mandatory parameter had a wrong value and was ignored
*   a non-recognised parameter (not amongst available options) was ignored

### Recoverable error

Alert about something that prevented the request to be (fully) processed accordingly to the logic, and that might lead to unwanted behaviour and/or partial result.

*   The code encountered a forbidden scenario or something that might lead to inconsistency and skipped some part of the processing.
*   Completeness of the request might be affected.
*   Execution is not halted.

**Examples**:

*   bulk CRUD request providing a list containing some nonexistent/invalid class, field or id
*   method call with inconsistent/invalid parameter

### Fatal-error

Information about a situation that could not be recovered from

*   When possible, some feedback about the error is provided.
*   Execution of the script is halted.

**Examples**:

*   memory allocation problem
*   syntax error (during parsing or compilation)
*   name resolution, dependency injection or file inclusion failure.

## General considerations

### Two kinds of fatal-errors

Sometimes, the distinction between recoverable-errors and fatal-errors is a bit ambiguous. When it comes to fatal-errors, a distinction should be made between "_system code_" and "_user code_".

Indeed, some errors (`E_ERROR`, `E_CORE_ERROR`, `E_COMPILE_ERROR`, `E_PARSE`) can be raised before the script is given the chance to customise how to handle these. In such scenario, we just want the processing to :

1.  Silently stop
2.  Log some information about the situation
3.  Return an error response (typically HTTP 500)

But other situations, when something makes the processing of the request impossible, can as well be considered as "fatal-errors" (authentication failure, missing mandatory data, invalid URI, ...). In those cases, processing should not be halted but rather send an appropriate response containing some information about the current error(s).

### Debugging

As there might be a lot of output, logging debug messages comes with an additional I/O cost. So, most of the time, it can be useful to :

*   categorize debugging information (we might want to display debug details only for a section of the application stack: SQL queries, ORM operations, specific provider actions, ...)
*   disable all debug messages

## Strategy

The questions we want to address here are :

*   How to make sure all errors are properly handled and to build a response according to the expected format?
*   How to provide developer with accurate information to help him fix code mistakes or unexpected behaviour?

While keeping in mind that :

*   raw error messages should never be sent directly to `://stdout`
*   most scripts are controllers, and are therefore expected to return a properly formatted response

### Error handling scenarios

![](https://i.imgur.com/2CEZjCpl.jpg "Errors in HTTP context")  

*   Unhandled errors and exceptions should silently fail (no output except **HTTP 500** error) and generate some output to the error log.
*   Whatever occurs, errors raised in **user** code should always fallback to an HTTP response (with **HTTP 4xx** code and formatted accordingly to HTTP request headers) informing that an error occurred and providing some details about it.
    *   These errors should be expected to be quite common and therfore not logged until specified otherwise.
    *   Framework code should mostly return 400 errors (bad request) while controller code should be more specific (401 unauthorized, 409 conflict, 423 locked, ...)
*   When developing a new controller, most warnings and recoverable errors should be logged silently and made available through a dedicated service or script, and response should notify about request completion (**HTTP 200**).

### Dealing with PHP

We can take advantage of the internal PHP reporting mechanism :
```php
// disable output to ://stdout
ini_set('display_errors', 0);
// ask for raw text messages
ini_set('html_errors', false);    
// output fatal-errors messages to a custom (system) error log
ini_set('error_log', LOG_STORAGE_DIR.'/error.log');
// request reporting for all error levels
error_reporting(E_ALL);
```

Besides:

*   Inside **user-code**, we can use PHP available constants for custom error kinds definition while maintaining the priority order (because we can only use `E_USER_*` constants, we drop general and deprecation notices and consider them as warnings):
    *   `E_ERROR`: `QN_REPORT_FATAL`
    *   `E_USER_ERROR`: `QN_REPORT_ERROR`
    *   `E_USER_WARNING`: `QN_REPORT_WARNING`
    *   `E_USER_NOTICE`: `QN_REPORT_DEBUG`
*   fatal-errors (`QN_REPORT_FATAL`) inside **user-code** can be raised by throwing exceptions (when not caught, PHP default behaviour is to stop current script execution).
*   debugging can be done with an additional flag allowing to:
    *   force providing explicit backtrace description for fatal errors
    *   effectively display verbose messages
*   if no error occured in **system-code**, handling of all errors and uncaught exceptions can be overloaded with a dedicated `ErrorReporter` provider using a distinct file for reporting: `LOG_STORAGE_DIR.'/qn_error.log'`.

`ErrorReporter` is a simple Singleton with no dependencies, which hijack PHP default Error and Exception handlers:
```php
public function __construct(/* no dependencies */) {
    // assign a unique thread ID (using a hash apache pid and current unix time)
    $this->setThreadId(md5(getmypid().microtime()));
    set_error_handler(__NAMESPACE__."\Reporter::errorHandler");
    set_exception_handler(__NAMESPACE__."\Reporter::uncaughtExceptionHandler");
}
```

And provides 4 main methods :
```php
public function fatal($msg) {
    $this->log(QN_REPORT_FATAL, $msg, self::getTrace());
    die();
}

public function error($msg) {
    $this->log(QN_REPORT_ERROR, $msg, self::getTrace());        
}

public function warning($msg) {
    $this->log(QN_REPORT_WARNING, $msg, self::getTrace());        
}

public function debug($source, $msg) {
    $this->log(QN_REPORT_DEBUG, $source.'::'.$msg, self::getTrace());
}
```

Here is an excerpt of the `log` method:
```php
private function log($code, $msg, $trace) {
    // check reporting level 
    if($code <= error_reporting()) {
        ...
            // append error message to log file
            file_put_contents(LOG_STORAGE_DIR.'/qn_error.log', $error, FILE_APPEND);                        
    }
}
```

In order to retrieve information about the current PHP stack, we use `debug_backtrace`.
```php
private static function getTrace($depth=0) {
    // skip the reporter inner calls
    $limit = 3+$depth;
    $n = $limit-1;
    $backtrace = debug_backtrace(DEBUG_BACKTRACE_IGNORE_ARGS, $limit);
    // retrieve info from where the error was actually raised 
    $trace = $backtrace[$n-1];

    ...
        return $trace;        
}
```

That way, logging can be achieved accordingly to the above mentioned constraints, either by using the `trigger_error` function or by calling the `ErrorReporter` public methods:
```php
/* debugging */
trigger_error('QN_SQL'.'sending SQL query: $query', QN_REPORT_DEBUG);
// equivalent to 
$reporter->debug('QN_SQL', 'sending SQL query: $query');

/* warning */
trigger_error('Pay attention here', QN_REPORT_WARNING);
// equivalent to 
$reporter->warning('Pay attention here');  

/* recoverable error */
trigger_error('Something is wrong here', QN_REPORT_ERROR);
// equivalent to 
$reporter->warning('Something is wrong here');

/* fatal error */
throw new Exception(QN_REPORT_FATAL, 'Things have gone really bad, stopping');
// equivalent to 
$reporter->fatal('Things have gone really bad, stopping');
```

### Going further

To improve chances that unhandled situations never occur, here are a few strategies:

*   System-code (framework):
    *   testing units
    *   no-man's land (auto-generated code) for critical parts (classes definition, schema creation, controller boilerplate)
*   User-code
    *   code quality check
    *   take advantage of the available Providers and Services (data validation, data adaptors, PHP context provider, ...) : the typical place to raise Exceptions that should be catched/handled inside the controllers that use them.