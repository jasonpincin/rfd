## Mariposa integration

There are a number of cases where we've wanted for lifecycle hooks to be able to communicate information back to ContainerPilot, or to be otherwise able to alter the ContainerPilot environment for purposes of future handler events or the main application itself. Additionally, it's likely that we at Joyent will want the [RFD36](https://github.com/joyent/rfd/blob/master/rfd/0036/README.md) scheduler for Triton to be able to have an interface to communicate with containers and ContainerPilot itself, for which we can provide an example in ContainerPilot.

This control plane would be useful for event hooks as well. Telemetry sensor outputs will be consumed via a robust API instead of the existing brittle text scraping. Event hooks will also be able to set environment variables for other services and event hooks.

This proposal is to create an HTTP server for ContainerPilot that exposes a simple HTTP REST protocol. *TODO: we need to have a discussion about security here: TLS, authn/authz*

**`GetStatus GET /v3/status`**

This API will expose the state of all services associated with this ContainerPilot instance.

*Example Response*

```
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 328
{
  "services": [
    {
      "name": "nginx",
      "address": "192.168.1.100",
      "port": 80,
      "status": "healthy"
    },
    {
      "name": "consul_agent",
      "address": "localhost",
      "port": 8500,
      "status": "nonadvertised"
    }
  ]
}
```

**`PutEnviron POST /v3/environ`**

This API allows a hook to update the environment variables that ContainerPilot provides to lifecycle hooks. The body of the POST must be in JSON format. The keys will be used as the environment variable to set, and the values will be the values to set for those environment variables. The environment variables take effect for all future processes spawned. This API returns HTTP400 if the the key is not a valid environment variable name, otherwise HTTP200 with no body.

*Example Request*

```
curl -XPOST \
    -d '{"ENV1": "value1", "ENV2": "value2"}' \
    http:/v3/environ
```

**`PutMetric POST /v3/metric`**

This API allows a sensor hook to update Prometheus metrics. (This allows sensor hooks to do so without having to suppress their own logging, which is required under 2.x.) The body of the POST must be in JSON format. The keys will be used as the metric names to update, and the values will be the values to set/add for those metrics. The API will return HTTP400 if the metric is not one that ContainerPilot is configuring, otherwise HTTP200 with no body.

*Example Request*

```
curl -XPOST \
    -d '{"my_counter_metric": 2, "my_gauge_metric": 42.42}' \
    http:/v3/environ
```

**`Reload POST /v3/reload`**

This API allows a hook to force ContainerPilot to reload its configuration from file. This replaces the SIGHUP handler from 2.x and behaves identically: all pollables are stopped, the configuration file is reloaded, and the pollables are restarted without interfering with the services. This endpoint returns a HTTP200 with no body.

**`MaintenanceMode POST /v3/maintenance`**

This API allows a hook to force ContainerPilot to go into maintenance mode. This replaces the SIGUSR1 handler from 2.x and behaves identically: all health checks are stopped and the discovery backend is sent a message to deregister the services. This endpoint returns a HTTP200 with no body.

Related GitHub issues:
- [Expose ContainerPilot state thru telemetry](https://github.com/joyent/containerpilot/issues/154)
- [HTTP control plane](https://github.com/joyent/containerpilot/issues/244)
- [Ability to pass SIGUSR1 and SIGHUP to main app](https://github.com/joyent/containerpilot/issues/195)
- [SIGINT termination and coprocess suspension](https://github.com/joyent/containerpilot/pull/186)