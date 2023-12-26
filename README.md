# ed-php-routing
Building Simple PHP Router

## introduction:

In this article, we will explore how to build a simple router to handle routing in PHP web applications. A router is a crucial component that maps incoming requests to the appropriate controllers and methods, enabling you to create clean and organized code structures.

To follow along with the tutorial, we will be using Docker to set up our development environment. Docker allows for easy portability and reproducibility of the development environment across different systems.

Here’s a brief overview of what we will cover in this article:

1 Setting Up the Development Environment: We will start by creating a Dockerfile and a docker-compose.yml file to set up our PHP and Nginx containers. This will provide us with a consistent and isolated environment to develop our custom router.
2 Understanding the Basics of Routing: We will delve into the fundamentals of routing in PHP. Understanding these concepts is essential for building an effective router.
3 Designing the Router Class: We will design and implement the central router class that will handle incoming requests and route them to the appropriate controllers and methods. We will explore how to register routes, extract controller and method names from route actions, and invoke the corresponding methods.
4 Advanced Features and Considerations: We will discuss advanced features such as singleton design pattern and autoloading. These features enhance the capabilities of our application.

Setting up development environement:

Let’s start with the Dockerfile to build a docker image for PHP.

```
FROM php:8.2-fpm

RUN apt-get update && apt-get install -y \
    git \
    curl \
    zip \
    vim \
    unzip

WORKDIR /var/www
```

This Dockerfile sets up a PHP environment based on the php:8.2-fpm image, installs some commonly used packages, and sets the working directory to /var/www for the application files.

```
docker-compose.yml file:

version: '3.8'

services:
  app:
    build:
      context: ./
      dockerfile: Dockerfile
    container_name: php-app
    restart: always
    working_dir: /var/www/
    volumes:
      - ../src:/var/www
  nginx:
    image: nginx:1.19-alpine
    container_name: php-nginx
    restart: always
    ports:
      - 8000:80
    volumes:
      - ../src:/var/www
      - ./nginx:/etc/nginx/conf.d
```

This Docker Compose file defines two services: app for the PHP application container and nginx for the Nginx web server container. It configures the necessary settings for each container, including build options, restart policies, port mappings, and volume mounts. This setup allows the PHP application to be served by Nginx.

To start the containers, we run (docker compose up — detach).

Notice: be sure that you’re at the same directory of the docker-compose.yml when running the previous command.

In our docker\nginx\nginx.conf we specified that the root directory for the server is /var/www/public. So all our requests start from public/index.php file, so this file serves as the entry point for our application:

```
<?php

  require "../core/helpers.php";
  require "../core/Router.php";
  
  const BASE_PATH = __DIR__ . '/../';
  
  spl_autoload_register(function($class) {
  
      $class = str_replace("\\", DIRECTORY_SEPARATOR, $class);

      require basePath("{$class}.php");
  });
  
  
  $router = Router::getRouter();
  require BASE_PATH . "routes.php";
  
  
  $method = $_SERVER['REQUEST_METHOD'];
  $uri = $_SERVER['REQUEST_URI'];
  
  $router->route($method, $uri);
  
  exit();
```

Including Helper Files and Router Class:

- The script starts by requiring the helpers.php and Router.php files, which provide additional functionality and define the Router class.

Defining the Base Path:

- The script defines a constant BASE_PATH that represents the base directory path for the application.

Autoloading Classes:

- The spl_autoload_register() function is used to register an autoloader function for automatically loading classes when they are needed.
- The provided anonymous function takes a class name as a parameter.
- Finally, it includes the corresponding class file using the require statement and the basePath() helper function.

Creating the Router Instance:

- An instance of the Router class is obtained by calling the static method getRouter() on the Router class itself.
- The getRouter() method ensures that only one instance of the Router class exists by implementing the singleton design pattern.

Including Route Definitions:

- The script includes the routes.php file, which contains the route registrations and handlers.

Handling the Request:

- The script retrieves the HTTP method and URI from the $_SERVER superglobal array.
- The route() method of the $router object is called, passing the method and URI as parameters.
- The route() method matches the requested route based on the method and URI and executes the corresponding controller and method.

Let’s move to our Router class:

```
<?php

class Router
{
    private static $router;

    private function __construct(private array $routes = [])
    {
    }

    public static function getRouter(): self {

        if(!isset(self::$router)) {

            self::$router = new self();
        }

        return self::$router;
    }
```

Class Properties:

- The class has a private static property $router , which will hold the singleton instance of the class.
- It also has a private property $routes , which is an array that will store the registered routes.

Constructor:

- The class has a private __construct(), which means that it can only be called from within the class.

Singleton Design Pattern Implementation:

- The class defines a static method getRouter() which returns an instance of the Router class. This method ensures that only one instance of the class exists throughout the application's lifecycle.
- Inside getRouter(), it checks if the $router instance is not set. If it is not it creates a new instance of the Router class using the private constructor and assigns it to the $router property.
- Subsequent calls to getRouter() will return the existing $router instance, ensuring that only one instance of the class is used.

```
public function get(string $uri, string $action): void {

    $this->register($uri, $action, "GET");
}

public function post(string $uri, string $action): void {

    $this->register($uri, $action, "POST");
}

public function put(string $uri, string $action): void {

    $this->register($uri, $action, "PUT");
}

public function delete(string $uri, string $action): void{

    $this->register($uri, $action, "DELETE");
}

protected function register(string $uri, string $action, string $method): void {

        if(!isset($this->routes[$method])) $this->routes[$method] = [];

        list($controller, $function) = $this->extractAction($action);

        $this->routes[$method][$uri] = [
            'controller' => $controller,
            'method' => $function
        ];
}

protected function extractAction(string $action, string $seperator = '@'): array {

       $sepIdx = strpos($action, $seperator);

       $controller = substr($action, 0, $sepIdx);
       $function = substr($action, $sepIdx + 1, strlen($action));

       return [$controller, $function];
}
```

Route Registration Methods:

- The class provides several methods (get(), post(), put(), delete()) to register routes for different HTTP methods.
- These methods accept a URI (the endpoint URL) and an action (a string representation of the controller and method to be invoked for that route).
- The methods call the register() method internally to store the route information in the $routes array.

```
public function route(string $method, string $uri): bool {

        $result = dataGet($this->routes, $method .".". $uri);

        if(!$result) abort("Route not found", 404);

        $controller = $result['controller'];
        $function = $result['method'];

        if(class_exists($controller)) {

            $controllerInstance = new $controller();

            if(method_exists($controllerInstance, $function)) {

                $controllerInstance->$function();
                return true;

            } else {

                abort("No method {$function} on class {$controller}", 500);
            }
        }

        return false;
}
```

Route Handling:

- The route() method is responsible for handling the incoming request.
- It takes the HTTP method and URI as parameters.
- It first checks if a route is registered for the given method. If not, it aborts 404.
- If a matching route is found, it retrieves the corresponding controller and method from the $routes() array.
- It checks if the specified controller class exists. If it does, a new instance of the controller is created.
- Finally, it calls the specified method on the controller instance.

Let’s now take a look of our routes registeration (routes.php):

```
<?php

$router->get("/", "controllers\HomeController@home");
$router->get("/about", "controllers\HomeController@about");
$router->get("/contact", "controllers\HomeController@contact");
$router->get("/dashboard", "controllers\DashboardController@index");
```

- The root route (“/”) is associated with the home() method of the HomeController class located in the "controllers" namespace. This means that when a user visits the root URL, the home() method of the HomeController class will be executed.
- The “/about” route is associated with the about() method of the HomeController class. This means that when a user visits the "/about" URL, the about() method of the HomeController class will be executed.

and so on..

in our HomeController:

```
<?php

namespace controllers;

class HomeController {

    public function home() {

        echo "Welcome to home page";
    }

    public function about() {

        echo "Welcome to about page";
    }

    public function contact() {

        echo "Welcome to contact page";
    }
}
```

We’re just echoing some text in our methods, but we can do alot more like (returnin a view, or some json response …).

Let’s move to our helpers file (core\helpers.php):

```
<?php

function basePath($path) {

    return BASE_PATH . $path;
}


function abort($message, $code = 404) {

    http_response_code($code);
    echo $message;
    exit();
}

function dataGet($arr, $key) {

    if (!is_array($arr) || empty($key)) {
        return null;
    }

    $keysArr = explode(".", $key);
    $searchedKey = $keysArr[count($keysArr) - 1];

    $i = 0;

    if(array_key_exists($keysArr[$i], $arr)) {

        $nextArr = $arr[$keysArr[$i]];

        while($i < count($keysArr)) {

            $i++;

            if(!array_key_exists($keysArr[$i], $nextArr)) break;

            if($keysArr[$i] === $searchedKey) return $nextArr[$keysArr[$i]];

            $nextArr = $nextArr[$keysArr[$i]];
        }
    }

    return null;
}
```

basePath($path):

- The purpose of this function is to provide a convenient way to generate absolute file paths by appending a relative path to the base path of the application.

abort($message, $code = 404)

- This function is responsible for handling error scenarios and terminating the script execution.

dataGet($arr, $key):

- This function is responsible for retrieving nested key from within a nested (key, value) array.

**Request lifecycle:**

1 The request is received by the web server (Nginx) listening on port 8000 (as specified in the dockr-compose.yml file).
2 The web server forwards the request to the PHP-FPM container running PHP 8.2.
3 The PHP script index.php is executed as the entry point for the request.
4 The getRouter() method of the Router class is called, which returns an instance of the Router class.
5 The routes.php file is included, which contains route registrations using the get() method of the Router instance.
6 The route “/” is registered with the HomeController@home action using the get() method of the Router instance.
7 When a request is made to the / endpoint, the route() method of the Router instance is called with the request method (e.g., "GET") and the request URI ("/") as arguments.
8 The home() method of the HomeController class is invoked, which echoes "Welcome to home page".
9 The PHP script execution ends, and the response is returned to the web server.
10 The web server sends the response back to the client making the request.

What extra features we can add?

1 Middlewares: Implement a middleware system to perform additional processing before or after the controller methods are executed.
2 Route Parameters: Extend the routing system to support dynamic route parameters, such as /users/{id}. This would allow capturing and passing parameters to controller methods for more flexible routing.

