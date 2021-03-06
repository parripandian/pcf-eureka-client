# pcf-eureka-client

JS implementation of a client for Eureka (https://github.com/Netflix/eureka), the Netflix OSS service registry. 
This is forked from Storytel/eureka-client (https://github.com/Storytel/eureka-client) to add OAuth2 related enhancements for Pivotal Cloud Foundry (PCF).
## Usage

First, install the module into your node project:

```shell
npm install pcf-eureka-client --save
```

### Add Eureka client to a Node application.

The Eureka module exports a JavaScript function that can be constructed.

```javascript
import Eureka from 'pcf-eureka-client';

// Or, if you're not using a transpiler:
const Eureka = require('pcf-eureka-client').Eureka;

// example configuration
const client = new Eureka({
  // application instance information
  instance: {
    app: 'some-service',
    hostName: 'localhost',
    ipAddr: '127.0.0.1',
    port: 8080,
    vipAddress: 'test.something.com',
    dataCenterInfo: {
      name: 'MyOwn',
    },
  },
  eureka: {
    serviceUrl: [
      'http://192.168.99.100:8761/eureka/v2/apps/',
      'http://192.168.99.100:32768/eureka/v2/apps/'
    ]
  },
  oauth2: {
      clientCredentials: {
          client_id: '<client_id>',
          client_secret: '<client_secret>',
          access_token_uri: '<access_token_uri>'
      }
  }
});
```

The Eureka client searches for the YAML file `pcf-eureka-client.yml` in the current working directory. It further searches for environment specific overrides in the environment specific YAML files (e.g. `pcf-eureka-client-test.yml`). The environment is typically `development` or `production`, and is determined by the `NODE_ENV` environment variable. The options passed to the constructor overwrite any values that are set in configuration files.

You can configure a custom directory to load the configuration files from by specifying a `cwd` option in the object passed to the `Eureka` constructor.

```javascript
const client = new Eureka({
  cwd: `${__dirname}/config`,
});
```

If you wish, you can also overwrite the name of the file that is loaded with the `filename` property. You can mix the `cwd` and `filename` options.

```javascript
const client = new Eureka({
  filename: 'eureka',
  cwd: `${__dirname}/config`,
});
```

### Register with Eureka & start application heartbeats

```javascript
client.start();
```

### De-register with Eureka & stop application heartbeats

```javascript
client.stop();
```

### Get Instances By App ID

```javascript
// appInfo.application.instance contains array of instances
const appInfo = client.getInstancesByAppId('YOURSERVICE');
```

### Get Instances By Vip Address

```javascript
// appInfo.application.instance contains array of instances
const appInfo = client.getInstancesByVipAddress('YOURSERVICEVIP');
```

## Configuring for AWS environments

For AWS environments (`dataCenterInfo.name == 'Amazon'`) the client has built-in logic to request the AWS metadata that the Eureka server requires. See [Eureka REST schema](https://github.com/Netflix/eureka/wiki/Eureka-REST-operations) for more information.

```javascript
// example configuration for AWS
const client = new Eureka({
  // application instance information
  instance: {
    app: 'some-service',
    port: 8080,
    vipAddress: 'test.something.com',
    statusPageUrl: 'http://__HOST__:8080/',
    healthCheckUrl: 'http://__HOST__:8077/healthcheck',
    dataCenterInfo: {
      name: 'Amazon',
    },
  },
  eureka: {
    // eureka server host / port / EC2 region
    host: 'eureka.test.mydomain.com',
    port: 80,
  },
});
```

Notes:
  - Under this configuration, the instance `hostName` and `ipAddr` will be set to the public host and public IP that the AWS metadata provides.
  - For status and healthcheck URLs, you may use the replacement key of `__HOST__` to use the public host.
  - Metadata fetching can be disabled by setting `config.eureka.fetchMetadata` to `false` if you want to provide your own metadata in AWS environments.

### Looking up Eureka Servers in AWS using DNS
If your have multiple availability zones and your DNS entries set up according to the Wiki article [Configuring Eureka in AWS Cloud](https://github.com/Netflix/eureka/wiki/Configuring-Eureka-in-AWS-Cloud), you'll want to set `config.eureka.useDns` to `true` and set `config.eureka.ec2Region` to the current region (usually this can be pulled into your application via an environment variable, or passed in directly at startup).

This will cause the client to perform a DNS lookup using the host of the currently selected server in `config.eureka.serviceUrl` and `config.eureka.ec2Region`. The naming convention for the DNS TXT records required for this to function is also described in the Wiki article above.

## Configuration Options
option | default value | description
---- | --- | ---
`logger` | console logging | logger implementation for the client to use
`eureka.heartbeatInterval` | `30000` | milliseconds to wait between heartbeats
`eureka.registryFetchInterval` | `30000` | milliseconds to wait between registry fetches
`eureka.fetchRegistry` | `true` | enable/disable registry fetching
`eureka.filterUpInstances` | `true` | enable/disable filtering of instances with status === `UP`
`eureka.ssl` | `false` | enable SSL communication with Eureka server
`eureka.useDns` | `false` | look up Eureka server using DNS, see [Looking up Eureka Servers in AWS using DNS](#looking-up-eureka-servers-in-aws-using-dns)
`eureka.fetchMetadata` | `true` | fetch AWS metadata when in AWS environment, see [Configuring for AWS environments](#configuring-for-aws-environments)
`eureka.useLocalMetadata` | `false` | use local IP and local hostname from metadata when in an AWS environment.

## Events

Eureka client is an instance of `EventEmitter` and provides the following events for consumption:

event | data provided | description
---- | --- | ---
`started` | N/A | Fired when eureka client is fully registered and all registries have been updated.
`registered` | N/A | Fired when the eureka client is registered with eureka.
`deregistered` | N/A | Fired when the eureka client is deregistered with eureka.
`heartbeat` | N/A | Fired when the eureka client has successfully renewed it's lease with eureka.
`registryUpdated` | N/A | Fired when the eureka client has successfully update it's registries.

## Debugging

The library uses [request](https://github.com/request/request) for all service calls, and debugging can be turned on by passing `NODE_DEBUG=request` when you start node. This allows you you double-check the URL being called as well as other request properties.

```shell
NODE_DEBUG=request node example.js
```

You can also turn on debugging within the library by setting the log level to debug:

```javascript
client.logger.level('debug');
```

## Known Issues

### 400 Bad Request Errors from Eureka Server

Later versions of Eureka require a slightly different JSON POST body on registration. If you are seeing 400 errors on registration it's probably an issue with your configuration and it could be the formatting differences below. The history behind this is unclear and there's a discussion [here](https://github.com/Netflix-Skunkworks/zerotodocker/issues/46). The main differences are:

- `port` is now an object with 2 required fields `$` and `@enabled`.
- `dataCenterInfo` has an `@class` property.

See below for an example:

```javascript
const client = new Eureka({
  // application instance information
  instance: {
    app: 'jqservice',
    hostName: 'localhost',
    ipAddr: '127.0.0.1',
    port: {
      '$': 8080,
      '@enabled': true,
    },
    vipAddress: 'jq.test.something.com',
    dataCenterInfo: {
      '@class': 'com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo',
      name: 'MyOwn',
    },
  },
  eureka: {
    // eureka server host / port
    serviceUrl: [ 'http://192.168.99.100:32768/eureka/v2/apps/' ],
  },
});
```

### 404 Not Found Errors from Eureka Server

This probably means that the Eureka REST service is located on a different path in your environment. The default is `http://<EUREKA_HOST>/eureka/v2/apps`, but depending on your setup you may need to set `eureka.servicePath` in your configuration to another path. The REST service could be hung under `/eureka/apps/` or possibly `/apps/`.

### Usage with Spring Cloud

If you are using Spring Cloud you'll likely need the following settings:

  - Set the path of the serviceUrl to `/eureka/apps/`.
  - Use the newer style of the configuration [here](#400-bad-request-errors-from-eureka-server) or Spring Cloud Eureka will throw a 500 error.
  - Put single quotes around boolean `@enabled`. Unfortunately, a 500 error regarding parsing [seems to occur](https://github.com/jquatier/eureka-js-client/issues/63) without that.

Below is an example configuration that should work with Spring Cloud Eureka server:

```javascript
const client = new Eureka({
  instance: {
    app: 'jqservice',
    hostName: 'localhost',
    ipAddr: '127.0.0.1',
    statusPageUrl: 'http://localhost:8080',
    port: {
      '$': 8080,
      '@enabled': 'true',
    },
    vipAddress: 'jq.test.something.com',
    dataCenterInfo: {
      '@class': 'com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo',
      name: 'MyOwn',
    },
  },
  eureka: {
    serviceUrl: [ 'http://192.168.99.100:32768/eureka/apps/' ],
  },
});
```

## Tests

The test for the module are written using mocha and chai. To run the unit tests, you can use the gulp `test` task:

```shell
gulp test
```

If you wish to have the tests watch the `src/` and `test/` directories for changes, you can use the `test:watch` gulp task:

```shell
gulp test:watch
```
