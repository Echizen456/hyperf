# Request object

 `Request object (Request)` is completely implemented based on the [PSR-7](https://www.php-fig.org/psr/psr-7/) standard and is implemented by [hyperf/http-message](https://github.com/hyperf/http-message) component provides implementation support.
 
> Note that [PSR-7](https://www.php-fig.org/psr/psr-7/) standard is `Request` designed `immutable mechanism`. All start with `with` return value of the method is a new object and will not modify the value of the original object
## Installation

This component is completely independent and suitable for any framework project.

```bash
composer require hyperf/http-message
```

 > If used in other framework projects, only the API provided by PSR-7 is supported. For details, you can directly refer to the relevant specifications of PSR-7. The usage described in this document is limited to the usage when using Hyperf.
 
## Get the request object

You can inject `Hyperf\HttpServer\Contract\RequestInterface` through the container to obtain the corresponding `Hyperf\HttpServer\Request`. The actual injected object is a proxy object, and the proxy object is the `PSR-7 request object (Request) for each request`, which means that this object can only be obtained during the ʻonRequest` life cycle. Here is an example of obtaining it:
```php
declare(strict_types=1);

namespace App\Controller;

use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Annotation\AutoController;

/**
 * @AutoController()
 */
class IndexController
{
    public function info(RequestInterface $request)
    {
        // ...
    }
}
```

### Dependency injection and parameters

If you want to obtain routing parameters through controller method parameters, you can list the corresponding parameters after the dependencies, and the framework will automatically inject the corresponding parameters into the method parameters. For example, your route is defined as follows:
```php
// Annotation method
/**
 * @GetMapping(path="/user/{id:\d+}")
 */
 
// Configuration method
use Hyperf\HttpServer\Router\Router;

Router::addRoute(['GET', 'HEAD'], '/user/{id:\d+}', [\App\Controller\IndexController::class, 'user']);
```

Then you can get the `Query` parameter `id` by declaring the `$id` parameter on the method parameter, as shown below:

```php
declare(strict_types=1);

namespace App\Controller;

use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Annotation\AutoController;

/**
 * @AutoController()
 */
class IndexController
{
    public function info(RequestInterface $request, int $id)
    {
        // ...
    }
}
```

In addition to obtaining route parameters through dependency injection, you can also obtain route parameters through the `route` method, as shown below:

```php
declare(strict_types=1);

namespace App\Controller;

use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Annotation\AutoController;

/**
 * @AutoController()
 */
class IndexController
{
    public function info(RequestInterface $request)
    {
        // If it exists, it returns, if it does not exist, it returns the default value null
        $id = $request->route('id');
        // If it exists, it returns, if it doesn't exist, it returns the default value 0
        $id = $request->route('id', 0);
        // ...
    }
}
```

### Request path & method

`Hyperf\HttpServer\Contract\RequestInterface` In addition to using the ʻAPIs` defined by the [PSR-7](https://www.php-fig.org/psr/psr-7/) standard, it also provides a variety of  Method to check the request, below we provide some examples of methods:
#### Get request path

The `path()` method returns the requested path information.  In other words, if the destination address of the incoming request is `http://domain.com/foo/bar?baz=1`, then `path()` will return `foo/bar`:
```php
$uri = $request->path();
```

The ʻis(...$patterns)` method can verify whether the incoming request path matches the specified rules.  When using this method, you can also pass a `*` character as a wildcard:

```php
if ($request->is('user/*')) {
    // ...
}
```

#### Get the requested URL

You can use the ʻurl()` or `fullUrl()` method to get the full ʻURL` of the incoming request.  The ʻurl()` method returns the ʻURL` without the `Query parameter`, and the return value of the `fullUrl()` method contains the `Query parameter`:
```php
// No query parameters
$url = $request->url();

// Bring query parameters
$url = $request->fullUrl();
```

#### Get request method

The `getMethod()` method will return the request method of `HTTP`.  You can also use the ʻisMethod(string $method)` method to verify whether the request method of `HTTP` matches the specified rule:
```php
$method = $request->getMethod();

if ($request->isMethod('post')) {
    // ...
}
```

### PSR-7 request and method

[hyperf/http-message](https://github.com/hyperf/http-message) The component itself is an implementation of [PSR-7](https://www.php-fig.org/psr/psr-  7/) Standard components and related methods can be called through the injected request object (Request).  If it is declared as [PSR-7](https://www.php-fig.org/psr/psr-7/) standard `Psr\Http\Message\ServerRequestInterface` interface during injection, the framework will automatically convert to the equivalent  The `Hyperf\HttpServer\Request` object in `Hyperf\HttpServer\Contract\RequestInterface`.
 > It is recommended to use `Hyperf\HttpServer\Contract\RequestInterface` to inject, so as to get the IDE's auto-complete reminder support for exclusive methods.
 
## Input preprocessing & normalization

## Get input

### Get all input

You can use the `all()` method to get all the input data in the form of an array:

```php
$all = $request->all();
```

### Get the specified input value

Use ʻinput(string $key, $default = null)` and ʻinputs(array $keys, $default = null): array` to obtain `one` or `multiple` input values ​​of any form:
```php
// return if it exists, return null if it doesn't exist
$name = $request->input('name');
// If it exists, it will return, if it does not exist, it will return to the default value Hyperf
$name = $request->input('name', 'Hyperf');
```

If the transmission form data contains data in the form of "array", you can use the "dot" syntax to get the array:

```php
$name = $request->input('products.0.name');

$names = $request->input('products.*.name');
```
### Get input from query string

Use the ʻinput`, ʻinputs` method to get the input data from the entire request (including the `Query parameter`), while the `query(?string $key = null, $default = null)` method can only get from the query string  Get input data:
```php
// return if it exists, return null if it doesn't exist
$name = $request->query('name');
// If it exists, it will return, if it does not exist, it will return to the default value Hyperf
$name = $request->query('name', 'Hyperf');
// If no parameters are passed, all Query parameters are returned as an associative array
$name = $request->query();
```

### Get `JSON` input information

If the requested `Body` data format is `JSON`, as long as the `Content-Type` `Header value` of the `Request object (Request)` is correctly set to ʻapplication/json`, you can use ʻinput(string $key  , $default = null)` method to access the `JSON` data, you can even use the "dot" syntax to read the `JSON` array:
```php
// return if it exists, return null if it doesn't exist
$name = $request->input('user.name');
// If it exists, it will return, if it does not exist, it will return to the default value Hyperf
$name = $request->input('user.name', 'Hyperf');
// Return all Json data as an array
$name = $request->all();
```

### Determine if there is an input value

To determine whether a certain value exists in the request, you can use the `has($keys)` method.  If the value exists in the request, it will return `true`, if it does not exist, it will return `false`. `$keys` can pass a string, or pass an array containing multiple strings. Only if all of them exist will it return `true`:
```php
// Only judge a single value
if ($request->has('name')) {
    // ...
}
// Judge multiple values ​​at the same time
if ($request->has(['name', 'email'])) {
    // ...
}
```

## Cookies

### Get Cookies from the request

 Use the `getCookieParams()` method to get all the `Cookies` from the request, and the result will return an associative array.
 
```php
$cookies = $request->getCookieParams();
```

If you want to get a certain `Cookie` value, you can get the corresponding value through the `cookie(string $key, $default = null)` method:
 ```php
// return if it exists, return null if it doesn't exist
$name = $request->cookie('name');
// If it exists, it will return, if it does not exist, it will return to the default value Hyperf
$name = $request->cookie('name', 'Hyperf');
 ```

## File

### Get uploaded files

You can use the `file(string $key, $default): ?Hyperf\HttpMessage\Upload\UploadedFile` method to get the uploaded file object from the request.  If the uploaded file exists, this method returns an instance of the `Hyperf\HttpMessage\Upload\UploadedFile` class, which inherits the `SplFileInfo` class of `PHP` and also provides various methods for interacting with the file:
```php
// If it exists, it returns a Hyperf\HttpMessage\Upload\UploadedFile object, if it does not exist, it returns null
$file = $request->file('photo');
```

### Check if the file exists

You can use the `hasFile(string $key): bool` method to confirm whether a file exists in the request:

```php
if ($request->hasFile('photo')) {
    // ...
}
```

### Verify successful upload

In addition to checking whether the uploaded file exists, you can also verify whether the uploaded file is valid through the ʻisValid(): bool` method:

```php
if ($request->file('photo')->isValid()) {
    // ...
}
```

### File path & extension

The ʻUploadedFile` class also contains methods to access the full path of the file and its extension.  The `getExtension()` method will determine the file extension based on the file content.  The extension may be different from the extension provided by the client:
```php
// The path is the temporary path of the uploaded file
$path = $request->file('photo')->getPath();

// Since the tmp_name of the uploaded file by Swoole does not retain the original file name, this method has been rewritten to obtain the suffix of the original file name
$extension = $request->file('photo')->getExtension();
```

### Store uploaded files

The uploaded file is stored in a temporary location before it is manually stored. If you do not store the file, it will be removed from the temporary location after the request is completed, so we may need to persist the file  For storage processing, use `moveTo(string $targetPath): void` to move temporary files to the location of `$targetPath` for persistent storage. The code example is as follows:
```php
$file = $request->file('photo');
$file->moveTo('/foo/bar.jpg');

// Determine whether the method has been moved by the isMoved(): bool method
if ($file->isMoved()) {
    // ...
}
```
