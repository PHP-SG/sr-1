
# PSG SR 1

```sh
composer require psg/sr1
```

```php
class Request implements Psg\Sr1\RequestInterface{}
```

# Implementations
-	[PHP-SG/sr-1-implementation](https://github.com/PHP-SG/sr-1-implementation)

# Changes Compared to FIG PSR 7
-	All classes now have factory methods and a "create" method:
	-	RequestInterface
		-	createRequest()
		-	create()
	-	ResponseInterface
		-	createResponse()
		-	create()
	-	ServerRequestInterface
		-	createServerReques()
		-	create()
	-	StreamInterface
		-	createStream()
		-	createFromFileStream()
		-	createFromResourceStream()
		-	create()
		-	createFromFile()
		-	createFromResource()
	-	UploadedFileInterface
		-	createUploadedFile()
		-	create()
	-	UriInterface
		-	createUri()
		-	create()

-	Request, Response have methods
	-	`withBodyString(string $body)` to set the body using a string
	-	`getBodyString()` to get the body as a string



# Problem With PSR 7, 15, 17
When I went through PSR 7, 15, and 17, it occurred to me that, in order to write middleware that replaced "bill" with "bob" in the body content of the HTTP response, I would need to do one of:
1.	depend on the framework to inject the factories and use __construct injection (which becomes a bit of a problem if my constructor takese other configuration parameters which aren't auto-injectable)
2.	require an implementation of PSR 17 factories within my middleware
3.	build my own implementation of a PSR 17 within my middleware

From the perspective of just wanting to replace "bill" with "bob", a effort that should take one line of code (preg_replace('/bill/', 'bob', $body)), I found these three solutions were not good.  Not only were they not good, but none of them were the standard, which means which one is selected, and how it is implemented, is up to the coder and may be variable.

Let's take a look at why these are bad solutions:
-	for #1, apart from making the assumption that the framework does IoC injection, it also means that if I had a middleware who's constructor looked like
```php
public function __construct(StreamFactory $streamFactory, UriFactory $UriFactory, $config_param1, $config_param2){}
```
That I would need to figure a way to inject my custom configuration before it was handed off to the framework to do automatic IoC injection.
-	For #2, this assumes the framework and user will import the middleware through composer, so that package dependencies are resolved.  I want people using my very simple middleware to have plug and play ability, without the need to run a composer dependency pull.  Should it be assumed that middleware is allowed to pull in external dependencies?  Some times yes, but in this simple middleware case, it makes no sense.
-	For #3, this is kind of a dumb solution.  Just to accomplish what is done is essentially one line of code, I have to spent who knows how long building my own, non-reusable implementation of PSR 17.

So, by this, I saw PSR 7 as flawed.  PSR 7 classes are already factories by the nature of them returning new instances of themselves upon modification, which means separation of concerns argument against making these classes full factories is void.

Finally, the convenience method of `withBodyString()`.  Most PHP responses are string bodies.  Yet, because there is the exception that some bodies are exceptionally long, FIG decided to throw out convenience and force everyone to handle the body as a stream; worse yet, a stream that you must import a factory for in order to revise in a meaningful way.
So, SR 1 has the convenience function, and relies on the implementation to determine how they want to handle making a stream from a string.
To parallel `withBodyString`, `getBodyString` is also provided just for expectation, but it is the same as `(string) $response->getBody()`.



# Why The Move Away From PSR 100-102

There was a problem related to PSR 100, and the subsequent 101 and 102.  By the nature of observing the liskov substitution principle, you can not provide an extension parameter in an extending interface.

For instance, with

```php
interface x{
	public function bob();
}
interface y extends x{
	public function bill();
}


interface Bob{
	public function process(x $param);
}
interface Sue extends Bob{
	public function process(y $param);
}
```

Can Sue fill in for Bob?  Sue may receive an `x` type param, and Sue may expect the functionality of a `y` type extension of `x`.  So, no, Sue may not be able to fill in for Bob.   As a consequence to this, the PSR 10X used original PSR 7 parameters.  The problem with this is it necessitates all interacting code to anticipate either PSR 100 or PSR 7 instances.  If code receives a PSR 7 response, say, from middleware that was produced under PSR 7, 15, then if the code wanted to use PSR 100 response features, it would have to upgrade the response to PSR 100, which is not with a short amount of code.

So, instead of this. I decided to rebuild these PSRs 100-102 into SR 1 (renamed at the behest of FIG).



