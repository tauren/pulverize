# pulverize - Micro-framework aware build tool

Tired of bloating your latest javascript library with the same code that is already included in many micro-frameworks
just because you don't want your library to have any dependencies? Add `pulverize` to your build process and let
your users decide what's included! 

Let's say your library needs `isFunction()` and `isString()`. Underscore provides these. So does jQuery. You know that
99% of the time, the users of your library will also be using underscore, jquery, zepto, YAHOO.lang, or whatever. But you don't know
which one, and you want to let people use it without any dependencies. 

If you are looking for production code, you'll have to come back later. This is currently just the beginnings of 
an idea and no code is available yet.

## The Idea

Code will be replaced inline during the build process. However, the raw, unprocessed code must be able to run without
building it first.

Tools to consider using:

* uglify
* smoosh
* commander.js

Will have a `pulverize` executable that will look for a `pulverize.json` configuration file in the current directory
when run. A path to the config file can be passed as a command line parameter. The config file will declare which
micro-framework combinations should be built for. A preliminary config file:

    {
      "VERSION": "0.0.1",
      "NAME": "mylib",
      "IN_DIR": "./src",
      "DIST_DIR": "./dist",
      "BUILDS": {
        "underscore": {
          libs: ["underscore"]
        },
        "jquery": {
          libs: ["jquery"]
        },
        "jqund": {
          libs: ["jquery","underscore"]
        }
      }
    }


The order libraries are specified in the config file is important and creates the precendence for which function 
from which framework is used. The config above would create the following files. There would also be an options 
to uglify the files as well.

    mylib-0.0.1.js
    mylib-underscore-0.0.1.js
    mylib-jquery-0.0.1.js
    mylib-jqund-0.0.1.js

Need to build an object containing the functions that you utilize in your library. Provide default implementations
for all of them. Override the defaults for various micro-frameworks. If an override is not specified, then the 
default will be included and used. Perhaps something like this?

    PULVERIZE = new Pulverize ['jquery','underscore'],
      defaults:
        isFunction: (obj) ->
          obj and obj.call and obj.apply
        isObject: (obj) ->
          cons = obj.constructor
          cons and 'Object' == cons.name
        isString: (obj) ->
          !!(obj == "" or (obj and obj.charCodeAt and obj.substr))    
      underscore:
        isFunction: _.isFunction
        isObject: _.isObject
        isString: _.isString
      jquery:
        isFunction: $.isFunction

This creates a `Pulverize` instance by passing an array of libs and an object containing the functions required by
your project. The constructor will add the appropriate functions to the PULVERIZE instance based on the array of 
libs passed to the contstructor.  Hopefully you will be able to call PULVERIZE whatever you want and that the
uglify AST will allow us to find `new Pulverize` and determine the name of the variable it is being assigned to.
This variable is needed later so that uses in your project code can be replaced.

Assume your project code is this:

    getUrl = (url) ->
      return url if PULVERIZE.isString url
      return url() if PULVERIZE.isFunction url
      throw 'Invalid URL'

And you build your project:

    pulverize

Perhaps allow command line options to build your project with specific frameworks?

    pulverize --lib jquery
    pulverize --lib jquery --lib underscore

The resulting code after pulverize is run in default mode would be:

    PULVERIZE = 
      isFunction: (obj) ->
        obj and obj.call and obj.apply
      isObject: (obj) ->
        cons = obj.constructor
        cons and 'Object' == cons.name
      isString: (obj) ->
        !!(obj == "" or (obj and obj.charCodeAt and obj.substr))    

    getUrl = (url) ->
      return url if PULVERIZE.isString url
      return url() if PULVERIZE.isFunction url
      throw 'Invalid URL'

However, if pulverize is run with both jquery and underscore, then the following would
be generated:

    getUrl = (url) ->
      return url if _.isString url
      return url() if $.isFunction url
      throw 'Invalid URL'

The order libraries are specified in the config file or on the command line will create
the precendence for which function is used. Because jquery is specified first, the
function `$.isFunction` is retained instead of `_.isFunction`.

Note that we have actually removed code from the output. If no default functions are
needed, then the entire PULVERIZE object is never included.

Will probably need to use uglify's AST to accomplish much of this.

## Concerns

The method signature of helper functions in different micro frameworks might vary. Need
to figure out a way to map from the default method signature to the method signature of
each micro-framework.

In other words, if framework A has `doSomething(obj,message)` and framework B has
`somethingTodo(message,obj)`, but they both accomplish the same thing, then we need a
way to map method signatures properly.

## Uses

I see this being helpful for I18N and L10N. For instance, the jQuery UI DatePicker has
functions to `formatDate()` and `parseDate`. There are other frameworks that have
similar functions. By using `pulverizer`, all of the code in your project to parse
dates could be excluded if the user is using jquery-ui, date.js, or such.

## Would be nice

Doing some kind of "feature detection" on the browser would be cool, but I'm not sure
how practical it would be. This would probably involve some sort of small "loader"
library that would detect what libraries were available on the client. It would then
request the appropriate version of your application based on this information. Again,
this needs to be thought through more, as the user could just make sure to put the
right version in the `<script>` tag.

