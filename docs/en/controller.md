# Controller

To process HTTP requests through the controller, you need to bind routing and controller methods in the form of `configuration files` or `notes`.
For details, please refer to the [Routing](zh-cn/router.md) chapter.   
For `Request (Request)` and `Response (Response)`, Hyperf provides `Hyperf\HttpServer\Contract\RequestInterface` and `Hyperf\HttpServer\Contract\ResponseInterface` to facilitate you to obtain input parameters and return data, about [Request]  For details of (zh-cn/request.md) and [response](zh-cn/response.md), please refer to the corresponding chapters.
 
## Write the controller

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use Hyperf\HttpServer\Contract\RequestInterface;
use Hyperf\HttpServer\Contract\ResponseInterface;

class IndexController
{
    // Obtain related objects by defining RequestInterface and ResponseInterface on the parameters, and the objects will be automatically injected by the dependency injection container
    public function index(RequestInterface $request, ResponseInterface $response)
    {
        $target = $request->input('target', 'World');
        return 'Hello ' . $target;
    }
}
```

> We assume that the `Controller` has passed the configuration file and defined the route as `/`, of course you can also use annotation routing

Call this address through `cURL`, you can see the returned content.

```bash
$ curl 'http://127.0.0.1:9501/?target=Hyperf'
Hello Hyperf.
```

## Avoid data confusion between coroutines

In the traditional PHP-FPM framework, it is used to provide an abstract controller or other named abstract parent class of Controller, and then the defined Controller needs to inherit it to obtain some request data or perform some return operations.  This is **cannot do** in Hyperf, because most of the objects in Hyperf, including `Controller`, exist in the form of `Singleton`, which is also for better reusing objects. The data related to the request also needs to be stored in the `coroutine context (Context)` under the coroutine, so please be careful not to store the data related to a single request in the class attribute when writing the code including non-static properties.   

Of course, if you have to store the request data through class attributes, there is no way. We can notice that we get the `Request (Request)` and `Response (Response)` objects by injecting `Hyperf\HttpServer\Contract\  RequestInterface` and `Hyperf\HttpServer\Contract\ResponseInterface` which are used to obtain them. Isn't the corresponding object also a singleton?  How is the coroutine safe here?  Take `RequestInterface` as an example. When the corresponding `Hyperf\HttpServer\Request` object gets the `PSR-7 request object`, it is obtained from the `coroutine context (Context)`, so the actual class used is only a proxy class which is actually called from the `coroutine context (Context)`.