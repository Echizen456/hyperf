# Dependency injection

## Introduction

Hyperf uses [hyperf/di](https://github.com/hyperf/di) as the framework's dependency injection management container by default. Although we allow you to replace other dependency injection management containers by design, we strongly recommend that you do not replace the component.   
[hyperf/di](https://github.com/hyperf/di) is a powerful component used to manage dependencies of classes and complete automatic injection. The difference from traditional dependency injection containers is that they are more in line with the long life cycle of the application uses, provides support for [annotation and annotation injection](zh-cn/annotation.md), and provides extremely powerful [AOP aspect-oriented programming](zh-cn/aop.md) capabilities. These capabilities and ease of use are as the core output of Hyperf, we confidently believe that this component is the best.

## Installation

This component exists by default in the [hyperf-skeleton](https://github.com/hyperf/hyperf-skeleton) project and exists as the main component. If you want to use this component in other frameworks, you can install it with the following command.

```bash
composer require hyperf/di
```

## Binding object relations

### Simple Object Injection

Generally speaking, the relationship and injection of the class do not need to be explicitly defined. Hyperf will do all this silently for you. We will illustrate the related usage through some code examples.     
Suppose we need to call the `getInfoById(int $id)` method of the `UserService` class in `IndexController`.
```php
<?php
namespace App\Service;

class UserService
{
    public function getInfoById(int $id)
    {
        // we assume there is an Info entity
        return (new Info())->fill($id);    
    }
}
```

#### Inject via constructor

```php
<?php
namespace App\Controller;

use App\Service\UserService;

class IndexController
{
    /**
     * @var UserService
     */
    private $userService;
    
    // Complete automatic injection by declaring the parameter type on the parameters of the constructor
    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }
    
    public function index()
    {
        $id = 1;
        // Use directly
        return $this->userService->getInfoById($id);    
    }
}
```

> Note that when using constructor injection, the caller, `IndexController`, must be an object created by `DI` to complete automatic injection, and `Controller` is created by `DI` by default, so you can directly use constructor injection

When you want to define an optional dependency, you can define the parameter as `nullable` or the default value of the parameter as `null`, which means that if the parameter is not found in the `DI` container or the corresponding object cannot be created, instead of throwing an exception, use `null` to inject.  *(This function is only available in 1.1.0 or higher)*

```php
<?php
namespace App\Controller;

use App\Service\UserService;

class IndexController
{
    /**
     * @var null|UserService
     */
    private $userService;
    
   // By setting the parameter to nullable, it indicates that the parameter is an optional parameter
    public function __construct(?UserService $userService)
    {
        $this->userService = $userService;
    }
    
    public function index()
    {
        $id = 1;
        if ($this->userService instanceof UserService) {
            // $userService is only available when the value exists
            return $this->userService->getInfoById($id);    
        }
        return null;
    }
}
```

#### Inject via `@Inject` annotation

```php
<?php
namespace App\Controller;

use App\Service\UserService;
use Hyperf\Di\Annotation\Inject;

class IndexController
{
    /**
     * Inject the attribute type object declared by the `@var` annotation through the `@Inject` annotation
     * 
     * @Inject 
     * @var UserService
     */
    private $userService;
    
    public function index()
    {
        $id = 1;
        // Use directly
        return $this->userService->getInfoById($id);    
    }
}
```

> Annotation injection via `@Inject` can be applied to objects created by `DI` and objects created by the `new` keyword;

> When using `@Inject` annotation, you need to `use Hyperf\Di\Annotation\Inject;` namespace;

##### Required parameters

There is a `required` parameter in the `@Inject` annotation. The default value is `true`. When the parameter is defined as `false`, it indicates that the member attribute is an optional dependency. When the object corresponding to `@var` does not exist in the `DI` container or cannot be created, it will not throw an exception but inject a `null`, as follows:

```php
<?php
namespace App\Controller;

use App\Service\UserService;
use Hyperf\Di\Annotation\Inject;

class IndexController
{
    /**
     * Inject the attribute type object declared by the `@var` annotation through the `@Inject` annotation
     * When UserService does not exist in the DI container or cannot be created, null is injected
     * 
     * @Inject(required=false) 
     * @var UserService
     */
    private $userService;
    
    public function index()
    {
        $id = 1;
        if ($this->userService instanceof UserService) {
            // $userService is only available when the value exists
            return $this->userService->getInfoById($id);    
        }
        return null;
    }
}
```

### Abstract object injection

Based on the above example, from a reasonable point of view, the `Controller` should not directly be an `UserService` class, but may be more of an `UserServiceInterface` interface class. At this time, we can use `config/autoload/dependencies.  php` to bind the object relationship to achieve the goal. Let's explain it through code.

Define an interface class:

```php
<?php
namespace App\Service;

interface UserServiceInterface
{
    public function getInfoById(int $id);
}
```

`UserService` implements interface class:

```php
<?php
namespace App\Service;

class UserService implements UserServiceInterface
{
    public function getInfoById(int $id)
    {
        // we assume there is an Info entity
        return (new Info())->fill($id);    
    }
}
```

Complete the relationship configuration in `config/autoload/dependencies.php`:

```php
<?php
return [
    \App\Service\UserServiceInterface::class => \App\Service\UserService::class
];
```

After this configuration, you can directly inject the `UserService` object through `UserServiceInterface`. We only use annotation injection as an example. The constructor injection is also the same:

```php
<?php
namespace App\Controller;

use App\Service\UserServiceInterface;
use Hyperf\Di\Annotation\Inject;

class IndexController
{
    /**
     * @Inject 
     * @var UserServiceInterface
     */
    private $userService;
    
    public function index()
    {
        $id = 1;
        // Use directly
        return $this->userService->getInfoById($id);    
    }
}
```

### Factory object injection

We assume that the implementation of `UserService` will be more complicated. When creating the `UserService` object, the constructor also needs to pass in some indirect injection parameters. Assuming we need to obtain a value from the configuration, then `UserService` needs to decide whether to enable the cache mode (by the way, Hyperf provides a more useful [model cache](zh-cn/db/model-cache.md) function) 

We need to create a factory to generate `UserService` objects:

```php
<?php 
namespace App\Service;

use Hyperf\Contract\ConfigInterface;
use Psr\Container\ContainerInterface;

class UserServiceFactory
{
    // Implement a __invoke() method to complete the production of the object, and the method parameters will be automatically injected into a current container instance
    public function __invoke(ContainerInterface $container)
    {
        $config = $container->get(ConfigInterface::class);
        // We assume that the key of the corresponding configuration is cache.enable
        $enableCache = $config->get('cache.enable', false);
        // make(string $name, array $parameters = []) method is equivalent to new, the use of make() method is to allow AOP to intervene, and directly use the new will cause AOP to fail to intervene in the process normally
        return make(UserService::class, compact('enableCache'));
    }
}
```

`UserService` can also provide a parameter in the constructor to receive the corresponding value:

```php
<?php
namespace App\Service;

class UserService implements UserServiceInterface
{
    
    /**
     * @var bool
     */
    private $enableCache;
    
    public function __construct(bool $enableCache)
    {
        // Receive the value and store it in the class attribute
        $this->enableCache = $enableCache;
    }
    
    public function getInfoById(int $id)
    {
        return (new Info())->fill($id);    
    }
}
```

Adjust the binding relationship in `config/autoload/dependencies.php`:

```php
<?php
return [
    \App\Service\UserServiceInterface::class => \App\Service\UserServiceFactory::class
];
```

In this way, when injecting `UserServiceInterface`, the container will be handed over to `UserServiceFactory` to create objects.

> Of course, in this scenario, the `@Value` annotation can be used to inject configuration more conveniently without building a factory class. This is just an example

### Lazy loading

Hyperf's long life cycle dependency injection is completed when the project starts. This means that long-lived classes need to pay attention to:

* The constructor is not yet a coroutine environment. If a class that may trigger the coroutine switch inject, the framework will fail to start.

* Avoid circular dependencies in the constructor (typical examples are `Listener` and `EventDispatcherInterface`), otherwise the startup will fail.

The current solution is: only inject `Psr\Container\ContainerInterface` into the instance, and other components are obtained through `container` when the non-constructor is executed.  However, PSR-11 states:
> Users should not pass the container as a parameter to the object and then obtain the dependency of the object in the object through the container. This is to use the container as a service locator, and the service locator is an anti-pattern

In other words, although this approach is effective, it is not recommended from the perspective of design patterns.

Another solution is to use the lazy proxy mode commonly used in PHP, inject a proxy object, and then instantiate the target object when it is used.  The Hyperf DI component is designed with lazy loading injection function.
Add the `config/autoload/lazy_loader.php` file and bind the lazy loading relationship:

```php
<?php
return [
    /**
     * The format is: agent class name => original class name
     * The proxy class does not exist at this time, and Hyperf will automatically generate this class in the runtime folder.
     * The proxy class name and namespace can be freely defined.
     */
    'App\Service\LazyUserService' => \App\Service\UserServiceInterface::class
];
```

In this way, when injecting `App\Service\LazyUserService`, the container will create a lazy loading proxy class and inject it into the target object.

```php
use App\Service\LazyUserService;

class Foo{
    public $service;
    public function __construct(LazyUserService $service){
        $this->service = $service;
    }
}
````

You can also inject lazy loading agents through the annotation `@Inject(lazy=true)`. Lazy loading is achieved through annotations without creating configuration files.

```php
use Hyperf\Di\Annotation\Inject;
use App\Service\UserServiceInterface;

class Foo{
    /**
     * @Inject(lazy=true)
     * @var UserServiceInterface
     */
    public $service;
}
````

Note: When the proxy object performs the following operations, the proxy object will be actually instantiated from the container.

```php
// method call
$proxy->someMethod();

// read attributes
echo $proxy->someProperty;

// write attributes
$proxy->someProperty = 'foo';

// Check if the attribute exists
isset($proxy->someProperty);

// delete attribute
unset($proxy->someProperty);
```

## Precautions

### The container only manages long-lived objects

Another way of understanding is that the objects managed in the container are all singletons. This design will be more efficient for long-life applications and reduce the creation and destruction of a large number of meaningless objects. This design is also means that all objects that need to be managed by the DI container cannot contain `state` values. 
`State` can be directly understood as values ​​that will change with the request. In fact, in [Coroutine](zh-cn/coroutine.md) programming, these state values ​​should also be stored in the `Coroutine context`,  That is `Hyperf\Utils\Context`.

## Short-lived objects

Objects created by the `new` keyword have undoubtedly short life cycles. What if you want to create a short life cycle object but want to use the `constructor dependency automatic injection function`?  At this time, we can use the `make(string $name, array $parameters = [])` function to create an instance corresponding to `$name`. The code example is as follows:   

```php
$userService = make(UserService::class, ['enableCache' => true]);
```

> Note that only the object corresponding to `$name` is a short-lived object, all dependencies of this object are obtained through the `get()` method, which is a long-lived object

## Get the container object

Sometimes we may wish to achieve some more dynamic requirements, we would like to be able to directly obtain the `Container` object, in most cases, the framework's entry class (such as command class, controller, RPC service provider) is created and maintained by `Container`, which means that most of the business code you write is under the management of `Container`, which means in most cases, you can obtain the `Hyperf\Di\Container` container object by declaring it in the `Constructor` or injecting the `Psr\Container\ContainerInterface` interface class via the `@Inject` annotation.  Demonstrate through code:

```php
<?php
namespace App\Controller;

use Hyperf\HttpServer\Annotation\AutoController;
use Psr\Container\ContainerInterface;

class IndexController
{
    /**
     * @var ContainerInterface
     */
    private $container;
    
    // Complete automatic injection by declaring the parameter type on the parameters of the constructor
    public function __construct(ContainerInterface $container)
    {
        $this->container = $container;
    }
}
```   

In some more extreme dynamic situations, or when it is not under the management of `Container`, if you want to get the `Container` object, you can also use `\Hyperf\Utils\ApplicationContext::getContaienr(  )` method to obtain the `Container` object.

```php
$container = \Hyperf\Utils\ApplicationContext::getContainer();
```
