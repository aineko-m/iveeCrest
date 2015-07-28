# iveeCrest
a PHP library for EVE Online's CREST API (beta)

Copyright © 2015 by Aineko Macx.
All rights reserved.


## License
Unless otherwise noted, all files in this distribution are released under the LGPL v3.
See the file LICENSE included with the distribution.


## Purpose and target audience
The goal of this project is to provide its users with a simple but powerful library to access EVE's CREST API. By hiding the complexities and some of the quirks of CREST, iveeCrest helps developers to quickly prototype scripts or develop full blown (web) applications. iveeCrest will likely be most useful for developers with at least basic PHP knowledge.

**As of july 2015 iveeCrest has fully been merged into [iveeCore](https://github.com/aineko-m/iveeCore/). All future development will take place in that project.**

## Features
- Object oriented design with high power-to-weight ratio
- Authenticated CREST based
- Methods for all endpoints reachable from CREST root are available, plus a few deeper endpoints. Easily expanded to support more (pull requests welcome!)
- Gathering of multipage responses
- Supports parallel GET requests with high performance asynchronous processing
- Multilayer cache design, supporting Memcached or Redis as caching backend
- The index-less collections returned by CREST are properly re-indexed by IDs
- Extensible via configurable subclassing
- A well documented and mostly PSR compliant codebase
- Includes a self-contained web-script to retrieve a refresh token


## Requirements
- PHP >= 5.4 (64 bit)
- php5-curl
- One of the following: Memcached + php5-memcached OR Redis + php5-redis ([phpredis](https://github.com/phpredis/phpredis))
- Developed and used under Ubuntu 14.04, other platforms might work but are untested

The necessary amount of cache memory to avoid hot data being evicted is strongly dependant on the usage pattern. Users should carefully monitor their caches (and review their applications data requirements) before hammering the CREST API with duplicate queries due to responses lost to memory pressure.


## Installation

You'll probably want to git clone iveeCrest directly into your project:

```
cd /path/to/my/project
git clone git://github.com/aineko-m/iveeCrest.git
```

### Getting a refresh token

To pull data from authenticated CREST, we'll need to acquire a refresh token, which is tied to your application and the character that authorized it's data to be used.

iveeCrest comes with a self-contained web-script that does just this. It can be found under www/getrefreshtoken.php.
Simply copy that file to a webserver and point your web browser to it. The script will take you through the steps. You'll be asked to register your application at [https://developers.eveonline.com/applications](https://developers.eveonline.com/applications) if you haven't already, selecting "CREST Access" and the "publicData" scope.

In the script form you'll then need to enter application ID, secret and redirect URL (filled automatically). When you submit the form you'll be redirected to the CREST authentication page where you'll need to authorize your app to access your characters data. With that done your browser will be redirected back to the script and it'll be able to pull the refresh token and display it. Take note, we'll need it.


### Setup iveeCrest

Make a copy of the file iveeCrest/Config_template.php, naming it Config.php and edit the configuration to match your environment.
This includes the CREST client details (CREST Base URL, client ID, client secret, refresh token), the http user agent and the cache configuration.
To switch between Memcached and Redis, apart from changing the port, you'll need to change the configured cache class. In the static array $classes find the key "Cache" and change it's value to "\iveeCrest\MemcachedWrapper" or "\iveeCrest\RedisWrapper".

That's about it. You can test the basic functionality by running the test/IveeCrestTest.php script with phpunit, which may take roughly 1 minute to run starting from a cold cache.


## Usage

Basic usage is shown with some examples:

```php
<?php
//initialize iveeCrest. Adapt path as required.
require_once('/path/to/iveeCrest/iveeCrestInit.php');

//instantiate the CREST client, passing the configured options
$client = new iveeCrest\Client(
    iveeCrest\Config::getCrestBaseUrl(),
    iveeCrest\Config::getClientId(),
    iveeCrest\Config::getClientSecret(),
    iveeCrest\Config::getUserAgent(),
    iveeCrest\Config::getClientRefreshToken()
);
//instantiate an endpoint handler
$handler = new iveeCrest\EndpointHandler($client);

//show response data from verifyAccessToken call
print_r($handler->verifyAccessToken());

//get regions endpoint
print_r($handler->getRegions());

//get specific region endpoint
print_r($handler->getRegion(10000002));

//gather all item groups (multipage response is gathered automatically)
print_r($handler->getItemGroups());

//get all market orders for Tritanium in The Forge
print_r($handler->getMarketOrders(34, 10000002));
```

These examples showcase some of the implemented endpoints in the EndpointHandler class. It is far from covering all endpoints, but more will be added over time. However, you are not forced to use it at all, you can also call the Client class directly and implement your own endpoint handling if it suits your use case better.

The Client class implements the infrastructure for getting data from CREST. This includes methods for authentication handling, simple requests to single endpoints, automatic gathering of multipage responses and parallel GET requests. The latter is a bit more complex to implement as callback functions need to be used to asynchronously process responses, but provides great flexibility and performance. EndpointHandler has a few examples of how to do it.

To help with the understanding of the structuring of the library, a [class diagram](https://github.com/aineko-m/iveeCrest/raw/master/doc/iveeCrest_class_diagram.pdf) is provided.


## Extending iveeCrest
If you extend the library with features that are generally useful and compatible with the goals and structuring of the project, Github pull requests are welcome. In any case, if you modify iveeCrest source code, you'll need to comply with the LGPL and release your modifications under the same license.

To adapt iveeCrest to your needs without changing the code, the suggested way of doing so is to use subclassing, creating new classes inheriting from the iveeCrest classes, and adapting the configuration (iveeCrest/Config::classes) accordingly. Class names are looked up dynamically, so with the adjustment objects from your classes will get instantiated instead.


## Developer notes
Although CREST is aimed at replacing not only the XML API but also the SDE, from the consumers perspective it has some weaknesses.
- Performance: Even with consumer side caching there is a general performance issue which makes live data pulling (as in "user waiting for the app") impractical except in [tailored cases](https://forums.eveonline.com/default.aspx?g=posts&t=402562). Due to the RESTful way the endpoints are structured, often dozens or hundreds of API calls are necessary to fetch just a single bit of wanted data.
- Inflexibility: CREST doesn't offer ways to filter, search or combine data like you can in a relational database, you generally have to pull in all the data in search yourself. This compounds on the performance issue.
- Availability: As CREST is tied to the live EVE cluster, it also follows the daily downtime. 3rd party API access is also among the first to be shut down in case of problems.

One of the possible solutions for these issues (independent of this library) is to collect and persist the data from CREST in a database and serve the application from there instead of pulling it live. That adds complexity to the application code but that's a price to pay for gaining resilience and ergonomics.


## Future plans
Both CREST and this library are unfinished, so significant changes should be expected with time. More endpoint handling methods will come. At some point CREST will introduce changes that will certainly make some reworking of the library necessary. More authorization scopes, for instance. iveeCrest will also be integrated into [iveeCore](https://github.com/aineko-m/iveeCore), but I intend to maintain this "solo" library in parallel.

If you find bugs, have any suggestions or other feedback or are "just" a user, please post in [this thread](https://forums.eveonline.com/default.aspx?g=posts&t=409103).


## Acknowledgements
EVE Online is a registered trademark of CCP hf.