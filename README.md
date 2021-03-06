Restful routing/controllers
=========

This is just a wrapper around [https://github.com/chriso/klein.php](Klein) that allows for 
easy construction of restful urls and controllers.

A url maps to a controller based on a routes mapping, and the controller handles the
http request/response. Controllers can respond to types of routes, one for resources and
another for collections (of the same resource).5

Routes
-

A route looks something like **/owners/[a:name]/dogs** or **/users/[:userid]/photos/[i:photoid]**

The value in square brackets is matched and passed to the controller as query parameters.
A request to **/owners/george/dogs?color=brown** becomes
```
name = george
color = brown
```

The value behind the colon is the parameter name, and the value in front is a validation type.

See [https://github.com/chriso/klein.php#routing]() for more about route syntax.

----

The router takes a parameter that is an array of routes mapped to a controller, like
```php
"routes" => array(
            # requests to /api/v1/hello handled by tag_tool_api_v1_hello
            "/login" => "login",
            #  a namespace URL. Children of this are appended.
            "/users/[i:userid]" => array(
                "/loginhistory" => "loginhistory",
                "/messages" => "messages",
                "/photos" => "photos",
                "/photos/[i:photoid]" => "photos"
            )
        )
```
where the key is the route path and the value a namespace. If instead of a string, the value is an array, then the routes in that array are nested under the parent path.
The router also takes a parameter defining a route namespace and another one defining the controller namespace.

If your domain is **www.mypets.com**, the route namespace **/api/v1**, and the controller namespace **/app/controllers** then routes generated by this are:

```
/api/v1/login                               => /app/controllers/login.php
/api/v1/users/[i:userid]/loginhistory       => /app/controllers/loginhistory.php
/api/v1/users/[i:userid]/messages           => /app/controllers/messages.php
/api/v1/users/[i:userid]/photos             => /app/controllers/photos.php
/api/v1/users/[i:userid]/photos/[i:photoid] => /app/controllers/photos.php
```

The http methods supported by each route are determined by looking at the controller. Routes are either handled as a collection or a resource. A resource acts on a single record (database row), and a collection acts on many records (database table). In the example above, requests to /api/v1/users/[i:userid]/photos would be handled by the controller as a collection, and /api/v1/users/[i:userid]/photos/[i:photoid] as a resource. This is determined by the convention of resource routes ending in an id/paramter (*[i:photoid]*) and collections ending in the name of the collection (*photos*).

Controllers
-
Controllers handle the http request. A handler method expects to be passed a $request and $response object. The default mapping is below, this can be overridden in the main router object. Controllers don't need to extend anything, they just need to implement any of these methods. Methods not implemented become HTTP 405 errors when requested.

Every controller method is passed a response and request object. Check https://github.com/chriso/klein.php/wiki/Api for documentation.

| HTTP method | Controller method | Expected action |
| ----------- | ----------------- | --------------- |
| GET (resource) | fetch() | get the requested item (no side effects) |
| PUT (resource) | update() | update the requested item (idempotent) |
| DELETE (resource) | delete() | delete the requested item (idempotent) |
| GET (collection) | index() | list all items matching the query (no side effects) |
| POST (collection) | index() | create a new item |
| PUT (collection) | bulkUpdate() | update a set of items (idempotent) |
| DELETE (collection) | deleteAll() | delete all items (idempotent) |

Controllers also support custom actions. If a controller has the public method 
```public function translate($request, $response)```, then a POST to 
**/api/v1/languages/translate?from=english&to=esperanto** will call the method translate on the languages controller. If the requested method does not exist on the controller, then the response 404's.

If dev mode is enabled, GET requests to custom actions are allowed as well.

Usage
-

Put this in an index.php file:


```php
<?php
$urlRoot = '/api/v1';
$routes = array(
    "/login" => "login",
    "/users/[i:userid]" => array(
        "/loginhistory" => "loginhistory",
        "/messages" => "messages",
        "/photos" => "photos",
        "/photos/[i:photoid]" => "photos"
    )
);
$controllerPrefix = "tag_admin_api_v1";

$router = new tag_routing_core(
    $urlRoot,
    $routes,
    $controllerPrefix
);
//defualt uses $_SERVER vars
$router->routeRequest();

//Or specify directly (useful for manual driving and tests)
$router->routeRequest($_SERVER['REQUEST_URI'], $_SERVER['HTTP_METHOD]);
```

It's probably better to put all that in a config file and load them in the index file.

Controller example:

```php
<?php
class tag_admin_api_v1_photos {
    // list all photos for a user
    public function index($request, $response) {
        $uid = $request->userid;
        $list = $_TAG->dao->album[$uid]->getAll();
        $response->json($list);
    }
    // get a single photo
    public function fetch($request, $response) {
        $uid = $request->userid;
        $pid = $request->photoid;
        $response->json($_TAG->photo[$uid][$pid]);
    }
}
```

Dependencies
-

Klein (for routing) https://github.com/chriso/klein.php
Phockito for testing
