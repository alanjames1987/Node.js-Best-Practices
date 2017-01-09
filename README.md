# Node.js Best Practices

## Intention

This is a list of best practices for writing robust Node.js code. It is inspired by other guilds such as [Felix Geisendörfer's Node Style Guide ](https://github.com/felixge/node-style-guide) and what is popular within the community.

These best practices should relate to performance and functional differences between various types of coding. They should focus heavily on giving detailed written explanations and coding examples when possible. They should not be a list of style preferences. Lists like that already exists and honestly style doesn't matter when it comes to functionality.

If you think something is not explained enough or you disagree with a point please open an issue and discuss it. Please be respectful and open to discussing the idea. I want as many people involved in creating these as possible.

## Table of Contents

* [Always Use Asynchronous Methods](#always-use-asynchronous-methods)
* [Never require Modules After App Initialization](#never-require-modules-after-app-initialization)
* [Save a reference to this Because it Changes Based on Context](#save-a-reference-to-this-because-it-changes-based-on-context)
* [Always "use strict"](#always-use-strict)
* [Validate that Callbacks are Callable](#validate-that-callbacks-are-callable)
* [Callbacks Always Pass Error Parameter First](#callbacks-always-pass-error-parameter-first)
* [Always Check for "error" in Callbacks](#always-check-for-error-in-callbacks)
* [Use Exception Handling When Errors Can Be Thrown](#use-exception-handling-when-errors-can-be-thrown)
* [Use module.exports not exports](#use-moduleexports-not-exports)
* [Use JSDoc](#use-jsdoc)
* [Use a Process Manager like upstart, forever, or pm2](#use-a-process-manager-like-upstart-forever-or-pm2)
* [Follow CommonJS Standard](#follow-commonjs-standard)

## Always Use Asynchronous Methods

The two most powerful aspect of Node.js are it's non-blocking IO and asynchronous runtime. Both of these aspects of Node.js are what give it the speed and robustness to serve more requests faster than other languages.

In order to take advantage of these features you have to always use asynchronous methods in your code. Below is an example showing the good and bad way to read files from a system.

The Bad Way reads a file from disk synchronously.

```js
var data = fs.readFileSync('/path/to/file');
console.log(data);
// use the data from the file
```

The Good Way reads a file from disk asynchronously.

```js
fs.readFile('/path/to/file', function (err, data) {
	// err will be an error object if an error occurred
	// data will be the file that was read
	console.log(data);
});
```

When a synchronous function is invoked the entire runtime halts. For example, above The Bad Way halts the execution of any other code that could be running while the file is read into memory. This means no users get served during this time. If your file takes five minutes to read into memory no users get served for five minutes.

By contrast The Good Way reads the file into memory without halting the runtime by using an asynchronous method. This means that if your file takes five minutes to read into memory all your users continue to get served.

## Never `require` Modules After App Initialization

As stated above you should always use asynchronous methods and function calls in Node. The one exception is the `require` function, which imports external modules.

Node.js always runs `require` synchronously. This is so the module being required can `require` other needed modules. The Node.js developers realize that importing modules is an expensive process and so they intend for it to happen only once, when you start your Node.js application. They even cache the required modules so they won't be required again.

To explain one of the problems imagine you had a module that took 30 minutes to load, which is unreasonable, but just imagine. If that module is only needed in one route handler function it might take some time before someone triggers that route and Node.js has to `require` that module. When this happens the server would effectively be inaccessible for 30 minutes as that module is loaded. If this happens at peak hours several users would be unable to get any access to your server and requests will queue up.

The second problem is a bigger problem but builds on the first. If the module you require causes an error and crashes the server you may not know about the error for several days, especially if you use this module in a rarely used route handler. No one wants a call from a client at 4AM telling them the server is down.

The solution to both of these problems is to always `require` modules at application startup rather than during execution of a function during runtime. Node.js will cache loaded modules within the variable you specify allowing for faster code execution during runtime instead of blocking during runtime and causing potential issues.

The Bad Way

```js
app.get('/',function(req,res,next){

	var _ = require('underscore');

	// sort data within the request body
	_.sort(someArray,function(item){
		// do something with the item
	});
});
```

The Good Way

```js
var _ = require('underscore');

app.get('/',function(req,res,next){

	_.sort(someArray,function(item){
		// do something with the item
	});

});
```

## Save a reference to `this` Because it Changes Based on Context

If you have a background with Java, ActionScript, PHP, or basically any language that uses the `this` keyword you might think you understand how JavaScript treats the same keyword. Unfortunately you would be wrong.

Let me tell you how `this` is determined [officially by ECMAScript](http://www.ecma262-5.com/ELS5_Section_11.htm).

> The this keyword evaluates to the value of the ThisBinding of the current execution context.

Basically that means that the value of the `this` variable is determined based on context, not encapsulation, as it is in other languages.

For example, if `this` is used inside a function, `this` references the object that invoked the function. That means that if you create a constructor function (basically a class in JavaScript) which then has methods attached to it, the `this` variable in those methods may not refer to the constructor function (class) they are inside of.

The above happens a lot in Node, but it might be hard to understand without seeing code.

In the code below `this` has two different values.

```js
function MyClass() {
	this.myMethod = function() {
		console.log(this);
	};
}

var myClass = new MyClass();
myClass.myMethod(); // this resolves as the instance of MyClass

var someFunction = myClass.myMethod;
someFunction(); // this resolves as the window in a browser and the global object in Node
```

The best way to solve this is to preserve `this` as another variable and then use that other variable instead. The most common variable names to use are `_this`, `that`, `self`, or `root`.

I personally like `_this` or `self` best because `_this` is easy to understand and `self` will be understood by anyone with Python or Ruby experience as both languages use `self` instead of `this`.

After making the changes your code should look like this.

```js
function MyClass() {
	var self = this;
	this.myMethod = function() {
		console.log(self);
	};
}

var myClass = new MyClass();
myClass.myMethod(); // self resolves as the instance of MyClass

var someFunction = myClass.myMethod;
someFunction(); // self also resolves as the instance of MyClass
```

`self` now always refers to the `MyClass` instance.

You could also call useful binding method, but it is less readable in some cases.

```js
function MyClass() {

	this.myMethod = function() {
	  console.log(this);
	};
}

var myClass = new MyClass();
myClass.myMethod(); // self resolves as the instance of MyClass

var someFunction = myClass.myMethod.bind(myClass);
someFunction(); // self also resolves as the instance of MyClass
```

## Always "use strict"

"use strict" is a behavior flag you can add to the first line of any JavaScript file or function. It causes errors when certain bad practices are use in your code, and disallows the use of certain functions, such as `with`.

Believe it or not but the best place I found to describe what JavaScript's strict mode changes is [Microsoft's JavaScript documentation](http://msdn.microsoft.com/en-us/library/ie/br230269(v=vs.94).aspx#rest).

## Validate that Callbacks are Callable

As stated before Node.js uses a lot of callbacks. Node.js is also weakly typed. The compiler allows any variable to be converted to any other data type. This lack of strict typing can become a big problem when passing parameters into functions for one reason.

Only functions are callable.

This means that if you pass a string to a function, that needed a callback function, your application will crash when it tries to execute that string.

This is obviously bad, but upon first blush there is no simple way to solve it. You could wrap the execution of all callbacks in try catch statements, or you could use if statements to determine if a callback has been passed in.

However, there is a simple way of validating that callbacks are callable which requires only one line of code, and it accounts for optional callbacks as well as checking data type.

```js
callback = (typeof callback === 'function') ? callback : function() {};
```

This determines if the callback is a function. If it's not a function for any reason it creates an empty function and sets the callback to be that function. This way all callbacks are callable and optional.

Place that line at the top of each function that receives a callback and you will never crash due to uncallable callbacks again.

## Callbacks Always Pass Error Parameter First

Node.js is asynchronous, which means you usually have to use callback functions to determine when your code completes.

After writing Node.js code for a while you will want to start writing your own modules, which need callback functions to be passed in by the user of your module. If an error occurs in your module, how do you communicate that to the user of the module? If you're a Java developer you might think you should throw an exception, but throwing an exception in Node.js could potentially shutdown the server. Instead you should package the error into an object, and pass it to the callback function as the first parameter. If no error occurred you should pass null.

By [convention](http://nodeguide.com/style.html#callbacks) all callback functions are passed an error as the first parameter.

```js

function myFunction(someArray, callback){

	// an example of an error that could occur
	// if the passed in object is
	// not the right data type
	if( !Array.isArray(someArray) ){
		var err = new TypeError('someArray must be an array');
		callback(err, null);
		return;
	}

	// ... do other stuff

	callback(null, someData);

}

module.export.myFunction = myFunction;
```

## Always Check for "error" in Callbacks

As stated above, by convention an error is always the first parameter passed to any callback function. This is great for making sure your site doesn't crash and that you can detect errors when they happen.

Now that you know what they are you should start using them. If your database query errors out you need to check for that before using the results. I'll give you an example.

```js
myAsyncFunction({
		some: 'data'
	}, function(err, someReturnedData) {

		if(err){
			// don't use someReturnedData
			// it's not populated
			return;
		}

		// do something with someReturnedData
		// we know there was no error

	}
});
```

## Use Exception Handling When Errors Can Be Thrown

Most methods in Node.js will follow the "error first" convention, but some functions don't. These functions are not Node.js specific function, they instead come from JavaScript. There are lots of functions that can cause exceptions. One of these functions is `JSON.parse` which throws an error if it can't parse a string into JSON.

How do we detect this error without crashing our server?

This is a perfect time to use a classic JavaScript `try` `catch`.

```js
var parsedJSON;

try {
	parsedJSON = JSON.parse('some invalid JSON');
} catch (err) {
	// do something with your error
}

if (parsedJSON) {
	// use parsedJSON
}
```

You can now be sure that the JSON was parsed correctly before using it.

This can be even more useful when using it in modules.

```js
function parseJSON(stringToParse, callback) {

	callback = (typeof callback === 'function') ? callback : function() {};

	try {

		var parsedJSON = JSON.parse(stringToParse);

		callback(null, parsedJSON);

	} catch (err) {

		callback(err, null);

	}

}
```

Of course the above example is slightly contrived, however, the idea of using try catch is a very good practice.

## Use `module.exports` not `exports`

You might have used `module.exports` and `exports` interchangeably thinking they are the same thing and in many cases they are. However, `exports` is more of a helper method that collects properties and attaches them to `module.exports`.

So what's the problem? That sounds great.

Well don't get too excited. `exports` only collects properties and attaches them if `module.exports` doesn't already have existing properties. If `module.exports` has any properties, everything attached to `exports` is ignored and not attached to `module.exports`.

```js
module.exports = {};

exports.someProperty = 'someValue';
```

`someProperty` won't be exported as part of the module.

```js
var exportedObject = require('./mod');

console.log(exportedObject); // {}
```

The solution is simple. Don't use `exports` because it can create confusing, hard to track down bugs.

```js
module.exports = {};

module.exports.someProperty = 'someValue';
```

`someProperty` will be exported as part of the module.

```js
var exportedObject = require('./mod');

console.log(exportedObject); // { someProperty: 'someValue' }
```

## Use JSDoc

JavaScript is a weakly typed language. Any variable can be passed to any function without conflict, until you try to use that function.

```js
function multiply(num1, num2) {

	return num1 * num2;

}

var value = multiply('Some String', 2);

console.log(value) // NaN
```

Obviously this is a problem above that could easily be fixed by looking at the code. But what if the code was written by someone else and uses complex parameters that you don't really understand. You could spend several minutes tracking down the expected data type. Worst yet it might accept multiple data types, in which case it may take you longer to track it down.

The best thing to do is use [JSDoc](http://usejsdoc.org/). If you're a Java developer you will have heard of Javadoc. JSDoc is similar and at it's simplest adds comments above functions to describe how the function works, but it [can do a lot more](http://en.wikipedia.org/wiki/JSDoc#JSDoc_tags).

Some IDEs will even use JSDoc to make code suggestions.

## Use a Process Manager like `upstart`, `forever`, or `pm2`

Keeping a Node.js process running can be daunting. Simply using the `node` command is dangerous. If your Node.js server crashes the `node` command will not automatically restart the process.

However, programs like [upstart](https://en.wikipedia.org/wiki/Upstart), [forever](https://www.npmjs.org/package/forever), or [pm2](https://www.npmjs.org/package/pm2) will.

While upstart is a general purpose init daemon, forever and pm2 are specific to Node.

## Follow CommonJS Standard

Node.js follows a standard for writing code that varies slightly from the standards that govern writing browser based JavaScript.

This standard is called [CommonJS](http://wiki.commonjs.org/wiki/CommonJS).

While CommonJS is far too large for me to cover here it's worth knowing about and learning. The most important points are that it mandates certain file organization and behavior that should be expected from the CommonJS module loader (require). It also describes how internals of the Node.js system should work.

Check it out.
