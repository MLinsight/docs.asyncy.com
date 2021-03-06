# Storyscript

> A **Goal-oriented Cloud Native Programming Language** that choreographs microservices. By removing unnecessary complexity applications can be written with pure business logic that is intuitive, transparent and syntax-light.

# Table of Contents
[[toc]]

## About Storyscript

Storyscript focuses on the **application logic** rather than all the *tape and glue* that bind applications together. The underlining services have a standard for **logs, metrics, fail-over, rate-limiting, tracebacks and scaling** which eliminates the need to write it within the application. This cultivates a development environment primed for rapid application development in a production-ready platform.

Let's build a quick application for example. Our goals are to upload, analyse, compress and archive a video. A non-trivial application but in a **couple lines of Storyscript** we made it.

```coffeescript
# Registers with Asyncy Server as an endpoint
http server as client
  when client listen method:'post' path:'/upload' as client
      client write content:'Success! Processing asynchronously.'
      client set_status code:201
      client finish

      # At this we are running asynchronously

      # generate a unique id for this upload
      id = uuid uuid4

      video = client.files['my_uploaded_video']

      # using https://machinebox.io find the video topics
      topics = machinebox/videobox find_topics content:video

      # save record in mongodb
      mongodb insert db:'uploads' data:{'id': id, 'topics': topics}

      # using https://github.com/xiph/daala let's compress it to h264
      video = xiph/daala compress video:video codex:'h264'

      # upload to AWS S3
      aws/s3 put target:'/video/{{id}}.mp4' data:video
```

In comparison, the same application would likely take **hundreds of lines of code**, not to mention that each service above includes metrics, logging and scaling out-of-the-box.

<!-- > Blog: [Why Asyncy built a DSL called Storyscript](/) -->


## Syntax Overview

```coffeescript
###
Meet Storyscript
###

# Strings
my_string = "Hello"
"Say {{my_string}}!"  # string formatting
# Say Hello!

# Numbers
one = 1
onethree = 1.3

# Boolean
foo = true
bar = false

# List
letters = ['a', 'b', 'c']
letters[0]
# 1

# Object
fruit = {'apple': 'red', 'banana': 'yellow'}
fruit['banana']
# yellow

# Regexp
pattern = /^foobar/
('foobar' like pattern)
# true

# Files (provided by the service File)
file write path:'/folder/hello.txt' content:'hello world'
file read path:'/folder/hello.txt'
# hello world

# Date
birthday = date year:2018 month:1 day:1
tomorrow = (date now) + (interval days:1)

# Null
empty = null

# Conditions
if one > 1
    # then do this
else if one == 1
    # then do this
else
    # do this

# Loops
foreach my_list as item
    # more stuff here

while foobar
    # more stuff here

# Services
output = service cmd key:value

# Event-based service
slack bot as sb
    when sb hears pattern:/hello/ as msg
        msg reply message:'world'

# Functions
function walk distance:number returns string
    # more stuff here
    return "Ok, walked {{distance}}km!"

walk distance:10
# Ok, walked 10km!

# Chaining calls
my_service action foo:(my_string split by:',')
                  bar:(my_object find key:(my_list random))

# import another story
import 'folder/file' as my_function
# Call a method in that story
res = my_function key:value

# try and catch
try
  # more stuff here
catch as error
  # more stuff here
  retry  # try the block again
  # -or-
  raise  # bubble it up
```

## Semantics

### Procedure

```coffeescript
output = do_third foo:(do_second (do_first ...)) bar:(do_second ...)
```

Parentheses MUST be used to produce inline procedures. The innermost Parentheses will be executed first moving out to the outermost.

Same level parentheses MAY be called at the same time which done by parallel processing in new threads.

First set of parentheses when assigning variables is optional. E.g., `a = my_list length` is the same as `a = (my_list length)`.

### Attributes, methods and entries

Objects may have attributes and methods. A Map can *only* have entries.

1. `Object` attributes can **only** be accessed using a period character (`.`) as in `user.name`.
1. `Object` methods can **only** be accessed using a space character (` `) as in `cart add item:...`.
1. `Map` entries can **only** be accessed using brackets and strings (`['key']`) as in `some_map['key']`

The example below illustrates how the object `tweet` has attributes and methods.

```coffee
tweet = twitter tweet text:'Hello world'
id = tweet.id  # returns the tweet id 123456 of type int.
tweet like     # calls "like", a method of tweet
```

### Mutations

```coffeescript
my_list = []
my_list append item:1  # mutates my_list by appending the new item
(my_list == [1])
# true

my_string = 'abc'
my_string adjust replace:'a' with:'Z'  # does not mutate the original string
# Zbc
my_string
# abc
```

A variable MAY be mutated by it's type methods.

The comment, `Mutating Methods`, is added to examples below that identify methods as mutating the variable. This indicates a method that will mutate the variable and not requires reassignment.

### Variable Scope

Variables are not global.

```coffeescript
n = 1

every minutes:3
  n increment
  log n
```

```
+0  INFO 2
+3m INFO 3
+6m INFO 4
```

Functions do not have access to local variables. All variables must be provided as arguments.

```coffeescript
n = 1

function incr
  n increment
```

```
Syntax error. Variable "n" is not defined at line 4.
```

### Compiling

Storyscript is a dynamically compiled language. Type checking is performed at compile time, but not in a traditional way. From the perspective of the developer the following steps are performed during compile time.

Compile time consists of four primary processes:

1. **Linting** is performed to check syntax and grammar.
1. **Translation** is performed which translates the Storyscripts into event-logic tree.
1. **Dependancy** checks are performed to ensure command and arguments exists.
1. **Type-Checking** is performed on the Stories the ensure data integrity.

The type-checking includes the following checks:

1. Type mutation method exists.
2. Arguments are of the expected type.


### Environment Variables

Environment variables are stored in a restricted keyword `app.environment`.

```yaml
# asyncy.yml
environment:
  food: cake
  FOO: bar
```

```coffeescript
if app.environment.food == 'cake':
    ...

if app.environment.foo == 'bar':
    ...
```

::: tip Note

Variables are **ALWAYS** exposed as lower-case attributes.

:::

### Execution Model

Storyscripts are executed by an interpretation engine (not compiled to C or Java).

#### Deployment

1. All dependancies are gathered and prepared for execution.
1. The Asyncy Engine is prepared with the Stories as first-class assets for swift execution.
1. Every Storyscript is executed allowing them to register with the gateway, cron, etc.

#### Execution

A story may [execute in many ways](/faq/#how-are-storyscripts-started).

1. The Engine received notice to start a Story with or without starting arguments.
1. The Story is executed in a single thread.
1. When a service is called the Engine will communicate with the service passing necessary data to and from the service back into the primary thread.
1. Asynchronous commands may generate new threads and execute in the same pattern above.

```coffeescript
translated = service_a translate:my_string to:'spanish'
parts = translated split ' '
first_word = service_b name:parts[0]
```

The Story above is would perform the following operations:

1. Translate a string to Spanish
1. Split the translated string by whitespace
1. Assign `first_word` to the first word in the `parts` array


## Strings

```coffeescript
data = "foobar"

long_string = "Hi Friend,
This is a lo\
ng string."
# Hi Friend, This is a long string.

more_data = """
    The quick brown fox
    jumps over the lazy dog.
"""
# The quick brown fox\njumps over the lazy dog.

where = "Earth"
data_formatted = "Hello, {{where}}"
# Hello, Earth
```

::: v-pre
Like many traditional programming languages, Storyscript supports strings as delimited by the `"` or `'` characters.
Storyscript also supports string interpolation within "-quoted strings, using `{{ variable }}`.
Single-quoted strings are literal. You may even use interpolation in object keys.
:::

Multiline strings are allowed in Storyscript.
Lines are joined by a single space unless they end with a backslash.
Indentation is ignored.

Block strings, delimited by `"""` or `'''`, can be used to hold formatted or indentation-sensitive text (or, if you just don’t feel like escaping quotes and apostrophes).
The indentation level that begins the block is maintained throughout, so you can keep it all aligned with the body of your code.

Double-quoted block strings, like other double-quoted strings, allow interpolation.

### String Methods

```coffeescript
"abc" length
# 3

"abc" adjust replace:'b' with:'Z'
# aZc

"foo bar" capitalize
# Foo Bar

"foo bar" capitalize words:1
# Foo bar

"a,b,c" split by:','
# ['a', 'b', 'c']

"abc" uppercase
# ABC

"ABC" lowercase
# abc
```

## Numbers

```coffeescript
int = 1
number = 1.2
```

### Number Methods

```coffeescript
1 is_odd
# true

2 is_even
# true

-1 absolute
# 1

# Mutating Methods

n = 1
n decrement
# 0

n increment
# 1
```


## Comments

```coffeescript
###
Large
  Comment
Block
###

# Inline comment

foo = "bar"  # end of line comment
```

In Storyscript, comments are denoted by the `#` character to the end of a line,
or from `###` to the next appearance of `###`.
Comments are ignored by the compiler, though the compiler makes its best effort at reinserting your comments into the output JavaScript after compilation.


## Boolean

```coffeescript
happy = true
sad = false
```

## Lists

```coffeescript
list_inline = ["string", 1, 2]
list_multiline = [
  "string",
  1,
  2
]
```

### List Methods

```python
['a', 'b', 'c'] length
# 3

['a', 'b', 'c'] join by:':'
# a:b:c

['a', 'b', 'c'] index of:'b'
# 1

['a', 'b', 'c'] random
# c

# Mutating Methods

['a', 'b', 'c'] reverse
# ['c', 'b', 'a']

['a', 'b', 'c'] shift from:'left'
# a
# the list becomes ['b', 'c']

['1', '2', '3'] apply method:int
# [1, 2, 3]

['a', 'c', 'b'] sort dir:'asc'
# ['a', 'b', 'c']

my_list = [1, 2, 3]
[(my_list min), (my_list max), (my_list sum), (my_list reduce)]
# [1, 3, 6, -4]
# also try: average, mean, mode
```

Join a couple method in one line. `((('123' split) apply int) sum) == 6`

## Date, Internals and Ranges

```coffeescript
birthday = date year:2018 month:1 day:2
tomorrow = (date now) + (interval days:1)

r = range from:(date now) to:tomorrow
```

### Date Methods

```coffeescript
[bday year, bday month, bday day, bday hour, bday minute, bday second]
# [2018, 1, 2, 17, 32, 18]

bday format to:'YYYY-mm-dd'
# 2018-01-02
```

### Range Methods

```coffeescript
range days round:'down' # number of days within the range
# also try: year, months, days, hours, minutes, seconds
# round: down, nearest, up
```


## Objects

```coffeescript
object_inline = {'foo': 'bar', 'apples': 'oranges'}
object_multiline = {
  'foo': 'bar',
  'apples': 'oranges'
}
```

### Object Methods

```python
# access properies
foobar = {'key': 'value'}
foobar['key']
# value

{'a': 1, 'b': 2} length
# 2

{'a': 1, 'b': 2} keys
# ['a', 'b']

{'a': 1, 'b': 2} values
# [1, 2]

{'a': 1, 'b': 2} items
# [['a', 1], ['b', 2]]

# Mutating Methods

obj = {'a': 1, 'b': 2}
obj pop key:'a'
# 1
obj
# {'b': 2}
```


## Conditions

```coffeescript
if foo == bar
  # more stuff here
else if foo > bar
  # more stuff here
else
  # more stuff here

if (foo > 0 or cat is not dog) or foobar like /regexp/
  # more stuff here
```

`if`/`else` statements can be written without the use of parentheses and curly brackets. As with functions and other block expressions, multi-line conditionals are delimited by indentation.

## Looping

Looping through index and/or object keys.

```coffeescript
foreach my_list as item
    # ...

foreach my_list as index, item
    # ...

foreach my_object as key
    # ...

foreach my_object as key, value
    # ...

while foobar
    # ...
```

Loops have reserved keywords for ending and continuing loops.

```coffeescript
foreach my_list as item
    # more stuff here
    if do_end_loop
        end
    if do_skip_to_next_item
        next
    # ...
```


## Functions

```coffeescript
function get_user id:int returns map
    someone = (psql exec query:'select * from users where id={{id}} limit 1;')[0]
    someone['contact'] = fullcontact person email:someone['email']
    return someone

user_a = get_user id:7
user_b = get_user id:10
```

The example above is a function what queries the database and also downloads their FullContact profile.

Function must define their inputs and outputs which help with transparency, autocomplete and type checking during the Asyncy CI process.

Functions **MAY** declare one output and identify it's type. If an output type is used the function **MUST** use `return` and return that type.

```coffeescript
function add this:int that:int returns int
   return this + that
```

Functions that do not have an output may not use `return`. An error is thrown if a return is used.

```coffeescript
function do_this
    # ...

function do_that
    return 1
>>> ERROR: Function must set type of return.
```


## Services

```coffeescript
# Call a service with a command and all arguments named
service cmd key:value foo:bar

# Service output assigned to variable
foobar = service cmd key:value

# Arguments may by indented under the service
service cmd key:value
            foo:bar
```

In Storyscript, the syntax to run a service appears natural and arguments are named for transparency.

These services may be Docker containers that expose commands and define their interface. More details in [Services](/services/)

```coffeescript
tweet = "hello"
twitter tweet message:tweet
# would result in ```twitter tweet message:"hello"```
```

Containers, commands and argument names are **static grammar** and **interpreted literally**.

## Event-Based Services

Services may publish events which run a new block of logic.

```coffeescript
service cmd foo:bar as client
    when client event foo:bar as data
        # ...
```

A good example of this is streaming Tweets by hashtag.

```coffeescript
twitter stream as client
    when client tweet hashtag:'asyncy' as tweet
        res = machinebox/language data:tweet.message
        if res.tone == 'good'
            tweet reply message:'Thank you!'
            tweet retweet
            tweet like
```

Every new tweet will be passed into the block below in the variable `tweet`.
Then machine learning will determine if the tone of the tweet's message is good or bad. The streaming service will wait for new tweets until the story is ended.


## Importing

```coffeescript{1}
import 'utils/users' as users
# Call the function "get" which is defined in the Storyscript
res = users get key:value

# Or
import 'utils' as utils
res = utils/users get key:value
```

Import other Storyscripts by using the `import method from file` syntax.

The file path is **relative** to the Storyscript where the `import`. Use `/folder/...` for importing from the project root or `../folder` to import from the parent folder.

::: tip
The `.story` is optional. `/stories/users.story` is equivalent to `/stories/users`.
:::

```coffeescript
# foo.story
import 'bar' as Bar

# bar.story
import 'foo' as Foo
```

Stories MAY recursively import other stories, as seen above.


## Operations

```coffeescript{2}
if something_went_wrong
    end story
```

Use `end story` to stop the story and exit now.


## Exception Handling

```coffeescript
try
  # more stuff here
catch as error
  # more stuff here
finally
  # more stuff here
```

In Storyscript, the `try` expressions catch exceptions and pass the error to the `catch` block.

The `finally` block is **always** entered regardless of an exception being raised or not, use it for cleanup commands.

You may omit both the `catch` and `finally`.

```coffeescript
try
  # more stuff here
catch as error
  # more stuff here
  raise
```

Use the `raise` keyword to raise the exception, bubbling up to the next try block or stopping the story.


## Regular Expressions

```coffeescript
pattern = /^foo/i
```

Regular expressions are supported without any special characters of escaping necessary.

### Regular Expressions Methods

```coffeescript
pattern = /(?P<key>\w):(?P<value>\w)/
myString = 'foo:bar'

pattern matches str:myString
# true

pattern find in:myString
# {"key": "foo", "value": "bar"}

/(\w+)/ find in:'foo bar' many:true
# ['foo', 'bar']

/(?P<name>\w+)/ find in:'foo bar' many:true
# [{'name': 'foo'}, {'name': 'bar'}]
```

## Wait and Cron

The [`queue`](https://hub.asyncy.com/service/queue) comes complete with crons and delays.

```coffeescript
queue wait days:5 hours:2
    # do this in 5 days and 2 hours

queue wait date:((date now) + (interval day:1))
    # Hello, Tomorrow!

queue every hour:9
    # daily at 9am do this...

queue cron tab:'* * * * 9'
    # daily at 9am do this...
```

## Files

Asyncy provides access to a shared volume, unique to the Application. This volume should be treated as an ephemeral file storage, where contents are deleted at the end of the Story.

[`file`](https://hub.asyncy.com/service/file) is a service in the Asyncy Hub.

```coffeescript
file write path:'/folder/hello.txt' content:'hello world'
file read path:'/folder/hello.txt'
# hello world
```

## Types

```coffeescript
1 type
# int

true type
# bool

"" type
# string

[] type
# list

{} type
# object

null type
# null

(date now) type
# date

(interval days:1) type
# interval

(range from:foo to:bar) type
# range

/^foobar/ type
# regexp

function foobar returns int
    return 1

function foobar
    # ... no return allowed

foobar type
# function
```

Use the method `type` to get the type of a variable as a string.

```coffeescript
(1 is int) and (true is bool) and ("" is string)
# true

([] is list) and ({} is object)
# true

(1 is number) and (1.2 is number)
# true

{} is string
# false
```

Type checking can be applied to any type.
