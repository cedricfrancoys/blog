# About time

This article explores how to deal with timezones in a client-server context.

![](//i.imgur.com/r12bpUUl.jpg) 

## Time is relative...

Depending on the place we are using a web application, the current time it uses might vary up to 24 hours (and even 25 if you live in the Tonga islands or if you're working at the Amundsen-Scott polar station during the austral summer).

This is why we make a distinction between "local time" and "Coordinated universal time" (UTC or GMT).  
Local time is just the UTC with an offset matching the geographical area we're currently in: the "time zone" (e.g. UTC+1 or CET, UTC+3 or EAT, UTC-8 or PST)

So, when referring to a time or a date in a worldwide context, it is mandatory to use the same reference (i.e. UTC).

Ideally all data should be stored in a format that holds information about the timezone, such as the ISO 8601 (e.g. 1978-05-10T14:15:00+01:00)

For historical reason, some format don't hold that information (e.g. SQL `date` and `datetime` types) and thus rely on the server configuration to convert the stored dates and times into UTC.

## Client-server

When displaying dates and times through an App it has to be relatively to local time. When sending data to the server, the client has to use a format that holds the timezone offset.

### Client time to Server time conversion

    time = client_time - client_offset + server_offset

### Server time to UTC conversion

    time = server_time - server_offset

Reciproquely, on the server, when a date value is exported to be available for the client, it has to be in UTC.

## UNIX epoch

To deal with time measurements, the UNIX architecture started using timestamps: _the number of sixtieths of seconds elapsed since January 1 1971 00:00:00, represented as a 32-bits integer_ (as stated in the _"Unix Programmer's Manual"_, 1971).

To deal with cross-timezones networks and leap years, in 1988, the POSIX.1 redefined the timestamp as _the number of elapsed seconds since January 1 1970 00:00:00 GMT_ (GMT = UTC).

Of course this notation is less readable by humans and require conversions for representation of civil time.

But, most modern languages offer functions that deal with timestamps:

*   convert a timestamp to a human readable date
*   produce a timestamp based on an ISO date

both, accordingly to the timezone set in the configuration. Which, in turn, can easily be defined or re-defined (with PHP: `date_default_timezone_set()` or by setting date.timezone parameter in php.ini)

For consistency and logic this should be set accordingly to server configuration in `/etc/timezone` (used to set the TZ environment variable). However there might be exception to that rule.

## Format juggling

To deal with date format juggling, an advantageous solution is to define a **DataAdapter** Service for handling conversions between time formats that provides a `getMethod` method returning the appropriate function to convert a date/time to another syntax/variable.

most common syntaxes will be

*   native type of script language (e.g. "PHP")
*   JSON syntax
*   SQL syntax

Here is an excerpt of `DataAdapter::getMethod()` for the `datetime` type:
```php
'json' => [
    'php' =>
        // convert an ISO 8601 string to a timestamp
        function($datestr) {
            return strtotime($datestr);    
        }        
],
'php' => [
    'json' => 
        // convert a timestamp to an ISO 8601 string
        function($timestamp) {
            return date("c", $timestamp);
        },
    'sql' =>
        // convert a timestamp to an SQL datetime
        function($timestamp) {
            return date('Y-m-d H:i:s', $value);
        }
],
'sql' => [
    'php' =>
        // convert an SQL date to a timestamp
        function($datesql) {
            return strtotime($datesql);
        }
]
```

## Conclusion

### Use timestamps

When possible, the best strategy consist of storing all dates and times in UTC.

### Use ISO 8601 dates

If, for any reason, timestamps are not an option, [ISO dates (ISO8601)](https://en.wikipedia.org/wiki/ISO_8601) have to be preferred. Indeed, in most situations, they can safely be interpreted by the client and the server.

### Adapt dates between client, server and datastore

If dates are relative to server time, in case data are migrated to another server which uses a distinct timezone, in order to prevent data accuracy loss the new server has to be set up accordingly to the original configuration.

### Avoid data types that do not hold a full time description

In any case, some types should be avoided (such as native SQL `DATE` type), because it implies a precision loss when crossing timezones.

