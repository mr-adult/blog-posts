# JavaScript is Weird
///2024-05-04 Some interesting gotchas that can be found when deep diving into JavaScript.

JavaScript, as we all know is a quirky language. I've learned quite a lot in my 3 years using it and my time experimenting with parsing it. Here are some interesting ways you can break your favorite JavaScript linter or syntax highlighter as well as your website.

Before reading on, what do you think would be printed out given the following code block?

```javascript
// 1.
console.log("test".match(/test/));

// 2.
console.log(017 !== 15);
console.log(018 !== 18);

// 3.
// NOTE: 0041 is the hex code unicode code point for 'A'
let \u0041dam = "test";
Adam = "another string";
console.log(\u0041dam); 

// 4.
console.log(typeof null == "null");
console.log(null == undefined);

// 5.
console.log(null >= 0);
console.log(null > 0);
console.log(null > 0 || null == 0);

// 6.
const stringValue = "my string" // notice the lack of semicolon
[1, 2, 3].toString();
console.log(stringValue);

// 7.
let a = 1;
String.prototype.toString = a.__proto__.toString;
console.log("test".toString());
```

## JavaScript Has 2 Modes

JavaScript has some extra quirky behaviors that the standards committees decided to disallow in certain modes of JavaScript. If you've ever seen `"use strict"` or `'use strict'` at the top of a .js file, it's a cue to the engine that strict mode should be used. This will come up in some of the other sections.

## Regex is Supported Natively

```javascript
"test".match(/test/);
// is equivalent to 
"test".match("test");
// both of these evaluate to the array ["test"]
```

Maybe you've never seen it done this way, but JavaScript natively supports regex as part of its grammar. Creating your regex this way allows you to avoid double-escaping out of both the string syntax and regex syntax. To start and end a regex, simply use '/' around the regex.

```javascript
const str = "This \r\n is \r\n some \r\n text \r\n that \r\n uses \r\n windows \r\n newlines.\r\n";
str.match(/\r\n/);
// is equivalent to
str.match("\\r\\n");
```

## Legacy Octal Integer Literals

```javascript
const octal = 017; // This is evaluated to 15!
const nonOctal = 018; // but this is evaluated to 18!
console.log(017 !== 15); // -> false
console.log(018 !== 18); // -> true
```

The octal notation that JS uses is unfortunately inherited from the C programming language. Go test this in your favorite language. If it comes from before the year 2000 it probably has this behavior too (looking at Java, C, etc.). This behavior differs between strict and non-strict modes. If you prefix a number with a zero and all digits in the integer are valid octal digits (0-7), the number will be interpreted as an octal number.

```javascript
// In this snippet, we get an error from the JS engine 
"use strict";
const octal = 017; // -> Error
```

## Unicode Escapes are Allowed in Variable Names

```javascript
let \u0041dam = "test";
Adam = "another string";
console.log(\u0041dam); // -> "another string"
```

JS allows unicode escape sequences to be used in variable names. The JS engine will resolve them to their character value in the identifier (ex. 0041 is the unicode code point for 'A', so the engine will resolve '\u0041' to 'A'). The above code would print out "another string".

## Null Shenanigans

```javascript
console.log(typeof null == "null"); // -> false
console.log(null == undefined); // -> true

console.log(typeof null) // -> "object"
```

I would be remiss if I didn't mention how null is handled in JS, but there's not much to say about the first couple. typeof null evaluating to object seems like it was a bug in the original JS engine, but there's no fixing it now without breaking tons of websites!

```javascript
console.log(null >= 0); // -> true 
console.log(null > 0); // -> false
console.log(null > 0 || null == 0); // -> false
```

This case is a little bit more interesting. '>=' and '<=' act similarly to '=='. They trigger type-coalescing behavior. '>' and '<' also trigger type-coalescing behavior, but to a lesser extent. In this case, >= and > cause null to be type-coalesced to 0. The fact that '>=' is evaluated differentlly than '>' && '==' is terribly confusing in this case. 

## Automatic Semicolon Insertion

```javascript
const stringValue = "my string" // notice the lack of semicolon
[1, 2, 3].toString();
console.log(stringValue); // -> "y"
// is equivalent to 
const stringValue = "my string"[1].toString();
console.log(stringValue); // -> "y"
```

If you weren't aware, JS will insert semicolons into your code during parsing. It is usually very effective at this, but sometimes it does the wrong thing. If a line starts with an array [], no semicolon will be inserted before the array is parsed. Even more confusingly, a comma-delimited list in this case will evaluate to only the first element in that list.

## Prototypes

```javascript
String.prototype.toString = Number.__proto__.toString;
console.log("test".toString()); // Uncaught TypeError: Number.prototype.toString called on incompatible string
```

If you haven't heard of prototypes in JS, it's the mechanism through which methods are tied to classes and inheritance is done. In this case, we are overwriting the global String toString() method with the global Number function. Why JavaScript lets you do this is a mystery to me, but interesting none the less.

## Conclusion

If you got any of the answers to these questions correct, you must be quite familiar with JavaScript. These were all meant to be trick questions. If you got them all incorrect, then take this as a learning experience and maybe it will explain the next bug you find! Another weird behavior to call out that you might actually run into in the wild is this:

```javascript
let a = 0;
let b = a || null;
console.log(b); // -> null
```

Because 0 is falsey, or-assigning into a value that is 0 can overwrite it. This has caused a couple bugs which I have personally fixed.
