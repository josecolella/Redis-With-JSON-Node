# Introduction to using Redis as a JSON store

The following is a document that provides information on storing JSON in Redis, and the different strategies that are available with advantages and disadvantages. The examples with utilize node and the Redis client to store JSON. The majority of this information comes from the amazing presentation by [@itamarhaber](https://github.com/itamarhaber)

## Why did I create this README?

To give people an understanding as to the different possibilities in using Redis as a JSON store and showing a working code that allows anyone to jump start their project. I will be querying a json endpoint and saving the json in three different ways:

**As a key- value, where the value will be the JSON:**

```
SET key value
```

> Advantages

* Data is store serialized. "BLOB" read/write

> Disadvantages

* Element access is impossible - entire bulk must be read

**As a hash, where the value will be a Javascript Object:** 

```sh
HMSET key obj
```

> Advantages

- Elements are accessible in O(1)

> Disadvantages:

* No native way to decode/encode to/from JSON, means a client side implementation

* Only String/Number types

  ​

**Using ReJSON, a module created within Redis, that fully supports the JSON data type.**

```
JSON.SET key accessor value
```

> Advantages:

* Full JSON support

* Works with any Redis client

> Disadvantages

* Serializing the JSON is "expensive"; transform the input JSON to a tree structure internally

* Higher memory overhead

## Requirements

*Download the ReJSON docker image that provides Redis with the JSON module* 

```
docker run --rm -d -p 6379:6379 --name redis-rejson redislabs/rejson:latest
```

* Dependencies

```sh
npm install axios redis bluebird
```

* index.js

```javascript
const redis = require('redis');
const bluebird = require("bluebird");
const axios = require('axios');
const client = redis.createClient();

bluebird.promisifyAll(redis.RedisClient.prototype);
bluebird.promisifyAll(redis.Multi.prototype);
```

Below I will be showing the three different strategies described above:

* Setting the JSON as a string. I would use this strategy is the data will not change and there is no necessity in changing anything in the JSON

```javascript
// Calling a JSON endpoint that contains a list of emoji with descriptions. Saving the return as a string in Redis
axios.get("https://gist.githubusercontent.com/oliveratgithub/0bf11a9aff0d6da7b46f1490f86a71eb/raw/ac8dde8a374066bcbcf44a8296fc0522c7392244/emojis.json").then((response) => {
  client.setAsync("emojis", JSON.stringify(response.data));
});
```

* Setting the JSON as a hash. The problem with using this strategy is that only String/Number types are supported. If you have a null, or array with null values, you will be responsible with processing the data before saving in Redis. With simple json, this may not be an issue, but if your JSON has many nested levels, the cleaning process can become complex; where you will end up having to use [libraries](https://github.com/hughsk/flat) to flatten the JSON, and then unflatten when retrieving from Redis.
  * Below we can see an example of a more complex JSON with a nested structure and null values.

```json
exampleJSON = { "address": 
  {"location": null, "person": [{"name": "Joe Smith", "age": null}, {"name": "John Doe", "age": null}]}
}

client.hmsetAsync("emojis-hash", exampleJSON);
```

> Node Redis will provide the following message. So all validation and formatting code need to be added client-side

!["warning provided by node redis"](https://api.monosnap.com/rpc/file/download?id=H6Ww0xPuTfdRSbKetaqHBu0brVxRpF)





* Using **ReJSON** to set the JSON object

```javascript
axios.get("https://gist.githubusercontent.com/oliveratgithub/0bf11a9aff0d6da7b46f1490f86a71eb/raw/ac8dde8a374066bcbcf44a8296fc0522c7392244/emojis.json").then((response) => {
  //Set the JSON
  client.sendCommandAsync("JSON.SET", ["emoji-json", "." , JSON.stringify(response.data)]).then(reply=> console.log(reply))
});
```

	> Using ReJSON, you can now query by keys, and you can also modify the object without having to read it back into Node

```javascript
//Query of the JSON
client.sendCommandAsync("JSON.GET", ["emoji-json", "."]).then(reply => console.log(JSON.parse(reply)))

//You can also query specific keys
client.sendCommandAsync("JSON.GET", ["emoji-json", ".emojis[0]"]).then(reply => console.log(JSON.parse(reply)))


//With ReJSON you can also modify the JSON structure
client.sendCommandAsync("JSON.SET", ["emoji-json", ".emojis[0]", '{"name":"Example"}']).then(reply => console.log(reply))
```

* Using npm libraries - rejson
There are also npm libraries that allow for easier integration with rejson by providing semantically easier API than utilizing `sendCommandAsync`. One such library is [rejson](https://github.com/stockholmux/node_redis-rejson). 
Below is an example of how to use the library to set json and get specific json keys from redis, allowing you to pull specific keys without having to read the entire json in memory.

```sh
╭─josecolella at MacBookAir in /redis-with-json-introduction
╰─λ node                                                                                                                                                              0 < 21:22:42
> const redis = require("redis");
undefined
> const rejson = require("redis-rejson")
undefined
> rejson(redis)
undefined
> const client = redis.createClient()
undefined
> client.json_set("message", ".", JSON.stringify({key: "Hello Redis!!"}))
true
> client.json_get("message", ".key", (err, payload) => {
... console.log(payload);
... })
true
> "Hello Redis!!"

```

## Conclusions

If your application json does not change over time, and you require a simple caching of JSON, the string key value store strategy. 

If you application json is not overly complex with nested arrays, and potential null values, the hash strategy is probably the best way to go. With access at constant time, this strategy wins out.

Finally, if your application json changes and you require support of complex JSON objects, where you can modify and access specific keys, ReJSON is the way to go.



### Contributing

If there is anything that can be improved in this document, or if you feel that something does not make sense, make sure to open an issue. This document was created to help people get started with JSON in redis, with code examples.

:octocat: 

## Additional Resources



- RedisLab Video: https://youtu.be/NLRbq2FtcIk

- RedisLab blog post on Redis as a JSON store: https://redislabs.com/blog/redis-as-a-json-store/

  ​
