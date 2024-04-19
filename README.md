# docker-environment

This project demonstrates how do you wire in a proxy server (in our case it's Browser Mob Proxy) so that it sits between two servers and is capable of doing things that a browser mob proxy server does (altering requests, dropping headers etc.,)

## Components Involved

The following components are involved in this exercise.

### Browser Mob Proxy (BMP)

We have a simple dockerised version of [BrowserMobProxy](https://github.com/lightbody/browsermob-proxy) published into Github Container Registry. 

The source code of the dockerised BMP is available [here](https://github.com/krmahadevan/bmp_docker)


When this container comes up, it basically opens up and is listening on 2 ports.

* `8080` - This is the HTTP Port on which the BMP server listens to and will respond back to the [REST](https://github.com/lightbody/browsermob-proxy?tab=readme-ov-file#rest-api) calls.
* `8081` - This is the port of the proxy server that gets created and is now readily available for usage.

### Worker application

This dockerised springboot app listens on port `10020` port and has a single HTTP GET api that looks like below

```http request
GET http://localhost:10020/worker/greet HTTP/1.1
```

This returns a json that would look like

```json
{
    "message": "Greetings from worker e62c2a66-36c7-4726-a552-baeb83980bef"
}
```

and also a custom header `X-Worker-Id:"88c4aff8-1964-46b7-b6f4-61a8262bdaf9"` apart from copying all the headers that were sent as part of the request.

The source code of the dockerised MasterApp is available [here](https://github.com/krmahadevan/master)

### Master application

This dockerised springboot app listens on port `10010` port and also has a single HTTP GET api that looks like below

```http request
GET http://localhost:10010/master/greet HTTP/1.1
```

along with a mandatory header named `"dragon-warrior": "kungfu-panda"`

This returns a json that would look like

```json
{
    "message": "Hello Greetings from worker e62c2a66-36c7-4726-a552-baeb83980bef",
    "dragonWarrior": "[Vary:\"Origin\", \"Access-Control-Request-Method\", \"Access-Control-Request-Headers\", X-Worker-Id:\"88c4aff8-1964-46b7-b6f4-61a8262bdaf9\", user-agent:\"Java/17.0.10\", host:\"worker:10020\", accept:\"text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2\", dragon-warrior:\"My-Custom-User-Agent-String 1.0\", Content-Type:\"application/json\", Transfer-Encoding:\"chunked\", Date:\"Fri, 19 Apr 2024 09:43:46 GMT\", Via:\"1.1 browsermobproxy\", \"1.1 browsermobproxy\"]"
}
```

### The complete execution with the `docker-compose.yml` file

Run the commands from the directory which contains the `docker-compose.yml` file available in this repository.


* `docker-compose up`

Now run the below command to enable API call interception as explained [here]()

```shell
curl -i -X POST -H 'Content-Type: text/plain' -d "request.headers().remove('dragon-warrior'); request.headers().add('dragon-warrior', 'My-Custom-User-Agent-String 1.0');" http://localhost:8080/proxy/8081/filter/request
```

So here we are instructing browser mob proxy via its http port `8080` that the proxy that runs on the port `8081` should start filtering requests using the Javascript snippet shared above. The javascript is basically altering the value of the header `dragon-warrior` and setting it to `My-Custom-User-Agent-String 1.0`

Now when you invoke the below HTTP Call against our **MasterApp** using

```shell
curl --location 'http://localhost:10010/master/greet' \
--header 'krishnan: mahadevan' \
--header 'rest-client: true'
```

you should see a response like below (The complete logs from Postman)

```
GET http://localhost:10010/master/greet: {
  "Network": {
    "addresses": {
      "local": {
        "address": "::1",
        "family": "IPv6",
        "port": 55897
      },
      "remote": {
        "address": "::1",
        "family": "IPv6",
        "port": 10010
      }
    }
  },
  "Request Headers": {
    "krishnan": "mahadevan",
    "rest-client": "true",
    "user-agent": "PostmanRuntime/7.37.3",
    "accept": "*/*",
    "postman-token": "73df4a0d-8436-4155-84d6-91d88415b7bb",
    "host": "localhost:10010",
    "accept-encoding": "gzip, deflate, br",
    "connection": "keep-alive"
  },
  "Response Headers": {
    "vary": [
      "Origin",
      "Access-Control-Request-Method",
      "Access-Control-Request-Headers"
    ],
    "x-worker-id": "6d07a07c-5915-4fdd-b759-8381fa00b5b4",
    "user-agent": "Java/17.0.10",
    "host": "worker:10020",
    "accept": "text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2",
    "dragon-warrior": "My-Custom-User-Agent-String 1.0",
    "transfer-encoding": [
      "chunked",
      "chunked"
    ],
    "date": "Fri, 19 Apr 2024 10:33:35 GMT",
    "via": [
      "1.1 browsermobproxy",
      "1.1 browsermobproxy"
    ],
    "content-type": "application/json",
    "keep-alive": "timeout=60",
    "connection": "keep-alive"
  },
  "Response Body": "{\"message\":\"Hello Greetings from worker bf889c6d-35e2-4a0e-b4d9-869e13842248\",\"dragonWarrior\":\"[Vary:\\\"Origin\\\", \\\"Access-Control-Request-Method\\\", \\\"Access-Control-Request-Headers\\\", X-Worker-Id:\\\"6d07a07c-5915-4fdd-b759-8381fa00b5b4\\\", user-agent:\\\"Java/17.0.10\\\", host:\\\"worker:10020\\\", accept:\\\"text/html, image/gif, image/jpeg, *; q=.2, */*; q=.2\\\", dragon-warrior:\\\"My-Custom-User-Agent-String 1.0\\\", Content-Type:\\\"application/json\\\", Transfer-Encoding:\\\"chunked\\\", Date:\\\"Fri, 19 Apr 2024 10:33:35 GMT\\\", Via:\\\"1.1 browsermobproxy\\\", \\\"1.1 browsermobproxy\\\"]\"}"
}
```

Notice how the proxy server changed the value of the header `dragon-warrior` from `kungfu-panda` to `My-Custom-User-Agent-String 1.0`

Well, that's it. This entire exercise was to show how to plugin a proxy server into a containerised environment and get it to work.

### Some additional useful commands

* `docker-compose stop` - Stops all the services that are listed in the `docker-compose.yml` file.
* `docker-compose rm` - Removes all the services that are listed in the `docker-compose.yml` file.
* `docker-compose pull` - Fetches the latest images for the services that are listed in the `docker-compose.yml` file.

