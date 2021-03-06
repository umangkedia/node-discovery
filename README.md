[![Build Status](https://travis-ci.org/umangkedia/node-service-discovery.svg?branch=master)](https://travis-ci.org/umangkedia/node-service-discovery)

**Service Discovery module for nodejs built on Zookeeper**

This is an extension to the nicely built [Zoologist](https://github.com/ph0bos/zoologist) module. This also enables dynamic service registration and de-registration with ZK. 
It has support for **watchers** and **periodic polling** on Zookeeper to get the latest node data. 
Has support for **healthchecks** so you can safely take the nodes Out of Rotation(OOR) and bring Back In Rotation(BIR) and it will be removed from clients list.

##Getting started
1. Start the client. Client emits 'connected' event. You can listen to this if required
2. Create a serviceInstance which will be your current node. Add healthChecks to it to enable OOR, BIR support
3. Create a serviceDiscovery instance and register your serviceInstance on ZK. Also define the refresh interval for polling dependent services from ZK.
4. Create instance providers to get the dependent services from ZK. It will automatically add watcher to it.
5. Get instance using instance providers.

**Note**:
1. Don't save a node. The getInstance query is **extremely fast** and **in-memory**. It doesn't make any remote calls


####Data Model
Following data is stored on ZK path (/basePath/serviceName/HOST:PORT)
```json
{
	"host": "localhost",
	"port": 31299,
	"nodeData": {
		"environment": "stage"
	},
	"healthcheckStatus": "healthy",
	"lastUpdatedTimeStamp": 1478246554601
}
```

####Health Check (Optional)
Define a healthCheck. It should contain isHealthy() method. You can add an array of healthChecks if required
```javascript
var healthCheck = function () {
                return {
                    isHealthy: function () {
                        return true; //add logic here for returning healthy/unhealthy status
                    }
                }};
```
## Installation

    npm install node-service-discovery --save

## Examples

### Service Discovery

```javascript
'use strict';

var Ranger                  = require('node-service-discovery').Ranger;
var ServiceInstanceBuilder  = require("node-service-discovery").ServiceInstanceBuilder;
var ServiceDiscoveryBuilder = require('node-service-discovery').ServiceDiscoveryBuilder;

//Create a Ranger Client
var rangerClient = Ranger.newClient(process.env.ZK_CONNECTION_STRING || '127.0.0.1:2181');

/** start emits 'connected' event. You can initialize following things after the event
 *  Eg: rangerClient.once('connected') { //initialize following code }
 */
rangerClient.start();


/**
 * Create the Service Discovery Instance
 *
 * refreshInterval is the interval to check for the updation of nodes in ZK.
 */
var serviceDiscovery = ServiceDiscoveryBuilder
        .builder()
        .client(rangerClient)
        .serviceInstance(serviceInstance)
        .basePath('services')
        .refreshInterval(5000)
        .build();

// Create the instance provider (selectionStrategy: 'RoundRobin' or 'Random')
var instanceProvider = serviceDiscovery
        .instanceProviderBuilder()
        .serviceName('dependent-service')
        .selectionStrategy('RoundRobin')
        .build()

//init the provider, it adds watcher, adds periodic polling and gets instance data for first time
instanceProvider.init(function(error) {
    //check for error here
});

// Get an instance. The returned value will have 'data' model specified above
instanceProvider.getInstance();
```

### Service Registration

```javascript

'use strict';

var Ranger                  = require('node-service-discovery').Ranger;
var ServiceInstanceBuilder  = require("node-service-discovery").ServiceInstanceBuilder;
var ServiceDiscoveryBuilder = require('node-service-discovery').ServiceDiscoveryBuilder;

//Create a Ranger Client
var rangerClient = Ranger.newClient(process.env.ZK_CONNECTION_STRING || '127.0.0.1:2181');

/** start emits 'connected' event. You can initialize following things after the event
 *  Eg: rangerClient.once('connected') { //initialize following code }
 */
rangerClient.start();

/**
 * Register a Service
 *
 * process.env.environment is whatever is the environment you are using
 * Additionally, you can also bind the health-checks with the service instance (healthcheck is optional)
 */
var serviceInstance = ServiceInstanceBuilder
        .builder()
        .host("localhost")
        .port(8080) //integer
        .environment(process.env.environment || 'development')
        .healthChecks([healthCheck])
        .name('my-service')
        .build();
 
serviceDiscovery.registerService(function(error) {
    //check for error
});
```

### Health Check change propagation

```javascript

'use strict';

/* Register service using above example */

/** Emit following event on serviceDiscovery instance when health of you app changes on OOR/BIR
 * This will update the correct status for serviceInstance in Zookeeper
 * isHealthy() method mentioned above should return corresponding value after health status change
 */
 
serviceDiscovery.emit('healthCheckChange');


```
