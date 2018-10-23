# Symfony container autowiring guide

## Typehint (almost) everything

Firstly, each class in the project that depends on other classes 
(for example a Game class depends on the Evaluator class) must define explicitly the type of the
dependency in its constructor. With typehinting, Symfony knows how to automatically construct a 
service (class) and its dependencies (auto-wiring). Of course, some classes depend on primitive types
for the instantiation of objects (e.g. `Player("Foo")`). Such classes cannot be autowired and need
a Factory service that exposes them to the rest of the services.

## Http kernel and YAML setup

 - Require the following packages to the project:
    * `composer require symfony/http-kernel`
    *  `composer require symfony/config`
    * `composer require symfony/yaml`
 
 - Create an `app` folder in the root dir of the project.
 
 - Inside `app`, create a new folder `config` and inside this create a `services.yaml` file.
 The yaml file sets up autowiring for the services/classes of the project. We expose only the 
 services that we are going to call directly in the bin file 
 (for example `Symfony\Component\Console\Application`) and our game command service(s).
 ```
 services:
     # default configuration for services in *this* file
     _defaults:
         autowire: true      # Automatically injects dependencies in your services.
         autoconfigure: true # Automatically registers your services as commands, event subscribers, etc.
         public: false       # Allows optimizing the container by removing unused services; this also means
                             # fetching services directly from the container via $container->get() won't work.
                             # The best practice is to be explicit about your dependencies anyway.
 
     # for the project include all the class files. Avoid the classes with constructors that accept primitive types.
     # Use factory classes (for example for players) to provide these objects
     Vasileios\PokerProject\:
       resource: '../../src/*'
       exclude: '../../src/{{GameUtility/{Card.php,Player.php}},{PokerHand/CardsOfAKind.php}}'
 
     # we expose only the services that we are going to get from the bin script using the container.
     # in this case, the game command classes and Symfony CLI App
 
     Vasileios\PokerProject\Commands\FiveCardDrawCommand:
       public: true
 
     Vasileios\PokerProject\Commands\HoldemCommand:
       public: true
 
     Symfony\Component\Console\Application:
       public: true
 ```
 
  - Inside `app`, create a class AppKernel.php with the following contents 
  (make sure that composer can find it in `composer.json`). The kernel loads the YAML we
  created in the previous step:
  ```
  <?php
  
  use Symfony\Component\Config\Loader\LoaderInterface;
  use Symfony\Component\HttpKernel\Kernel;
  
  final class AppKernel extends Kernel
  {
      /**
       * In more complex app, add bundles here
       */
      public function registerBundles(): array
      {
          return [];
      }
  
      /**
       * Load all services
       */
      public function registerContainerConfiguration(LoaderInterface $loader): void
      {
          $loader->load(__DIR__ . '/config/services.yaml');
      }
  
  }
  ```

## Writing the bin script

In the application bin script, our newly exposed services can be fetched using Symfony 
Container's `get()` method, which accepts the name of the service and magically creates it for
us, along with all the dependencies it needs:
```
#!/usr/bin/env php
<?php
require_once dirname(__DIR__) . '/vendor/autoload.php';

use Symfony\Component\Console\Application;

$kernel = new AppKernel("dev", true);
$kernel->boot();

$container = $kernel->getContainer();
$application = $container->get(Application::class);
$fiveCardGame = $container->get(\Vasileios\PokerProject\Commands\FiveCardDrawCommand::class);
$holdemGame = $container->get(\Vasileios\PokerProject\Commands\HoldemCommand::class);

$application->addCommands([$holdemGame, $fiveCardGame]);
$application->run();
```

Notice that instead of `now()`, we use the container to get service objects (`$container->get(...)`)
