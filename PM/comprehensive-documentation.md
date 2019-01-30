# Comprehensive documentation


This article aims to explore a way to document a project while minimizing the time required to maintain that doc and make it profitable to everyone: developers, project manager(s) and final users.

![](http://i.imgur.com/liKCzcol.jpg)

## Why does it matter ?

In an Agile context, it is frequent, if not the routine, to adjust a piece of software that has been developped earlier. Or even sometimes,to re-do something that has been undone at some point as consequence of the customer will.

And murphy's-law states that, most of the time, this happens just long enough after for you to have forgotten what and how it had been done.

But abviouly, and rightfuly the customer expects that change to be made quickly as it is (or was) already there.

A well documented project empowers any contributor to the project to:
* remember the setting, the environment and avoid 'small' adjustments that weigh down any manipulation
* give the customer a good image and maintain confidence by showing a high quality of follow-up.

Of course, it is often difficult to justify the extra cost required for documenting. Depending on the receptivity of the customer, you must: either bill it as such; or increase the hourly rate (approximately 10%) to cover the additional time required.



## Where to place to documentation ?

Where it is possible : Inside the code, inside README docs, inside external reports.

When the produced code allows it, using a syntax that can both be added as such to an external document AND being displayed at the output of a call is a big advantage.

Although this can be a source of redundancy and make the maintenance of the documentation a little heavier, it is always easier to update a documentation rather than to "write it later".


## Documentation as sub-project

A common syntax among the developer community is markdown (a minimalist markup language). 

Many tools allow you to use this syntax to produce structured documentation in a minimum amount of time. 

In another post, I present how to use Typora, Github and MkDocs to produce interactive documentation, allowing lightning fast updates and accessible online with a pleasant visual interface.

## Self-documenting code


Here is such example:


If the script fails to execute due to some missing parameter, the script outputs its full description, along with the expected parameters and their related format.

And f course, this description also appear in the source code, which makes the understanding of the script purpose and usage, much easier to understand and to remember !



```php
$params = announce([
'description'   => 'This is an example to show how to use the announce() method',
'params'        => [
    'a'  => [
        'description'   => 'Mandatory argument that has to be formated as an integer',
        'type'          => 'integer',
        'required'      => true
     ],
    'b'  => [
        'description'   => 'This is an optional string argument',
        'type'          => 'string'
     ],
    'c'  => [
        'description'   => 'Optional argument with a default value (will always be present)',
        'type'          => 'string',
        'default'       => 'test'
     ]     
]
]);
```