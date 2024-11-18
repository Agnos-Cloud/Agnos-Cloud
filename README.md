# Agnos Cloud

A cloud-agnostic platform for designing, deploying, and managing micro-services.

## Key Concepts

### Service

A service is a logical representation of a microservice in a manner that does not change based on the host machine or environment.
In other words, the way you programmatically interact with a service should remain the same even across different deployment environments or
cloud service providers.

The actual work in a service is done by a component.

A service may run multiple instances of a component to handle traffic spikes. This also implies that a service has a built-in load balancer.

A service can have different components for different environments.

A service can have different environment variables for different environments.

A service has a name with which it can always be referenced.

A service can have inputs.

A service can have outputs, including the URL.

A service can expose an interface. An interface is the set of actions a service exposes as well as the
schema for the expected inputs for those actions.

``` yaml
API: 0.1
version: 1

environments:
- test
- local
- staging
- prod

services:
- name: email-service
  version: 2
  extends: email-service@1
  actions:
    send_email:
      schema:
        to:
          type: string
          required: true
          format: string/email
          validator: validate-to.js
        message: string
    send_email_template:
      schema:
        to: string
        template: string
    send_bulk_email:
        to:
          type: array
          item: string
        message: string
    send_bulk_email_template:
        to:
          type: array
          item: string
        template: string
  deployments:
  - environment: local
    resource: sendgrid@v1
    instances: 2
    actions:
      send_email:
        endpoint: /send
        data:
          to: to
          message: html_body
      send_email_template:
        endpoint: /send-template
        data: transform1.js
      send_bulk_email: script1.js
      send_bulk_email_template: script2.js

resources:
- name: sendgrid
  version: v1
  source: https://hub.docker.com/r/sendgrid/sendgrid-python/
  sourceType: docker
```

### Component

A component is an executable package.

A component  can be created from an HTTP server, a Docker image, a Git repo, or a local folder.

A component can have persistent storage.

A component can expose a home page used to show overviews (e.g. components for logging, monitoring, and alerting)

### Referencing a Service

A service, ServiceA, can reference another service, ServiceB (as long as there's no cyclical dependence).
During deployment, ServiceB will be deployed before ServiceA, and the URL of ServiceB will be an environment
variable for ServiceA.

You can also reference a service directly using the SDK.


``` js
import services from '@agnos-cloud/services';

const serviceB = await services.getUrl('ServiceB');
```

If the URL cannot be obtained from the environment variable, it will be obtained from the Agnos Cloud server.

### Actions

An action is a message sent from one service to another. It is synchronous, meaning the response to the action is expected
in the same request-response cycle.

``` js
import { ActionDispatcher } from '@agnos-cloud/core';

const dispatcher = new ActionDispatcher();
const response = await dispatcher.dispatch({
  name: 'SEND_EMAIL',
  data: {
    to: 'example@email.com',
    message: 'Hello World!',
  },
});
```

Agnos will determine the right service to which an action is to be dispatched; if no service is found an error should be returned or thrown.

It should be possible to specify the name of a service to which the action must be sent.

``` js
import { ActionDispatcher } from '@agnos-cloud/core';

const dispatcher = new ActionDispatcher();
const response = await dispatcher.dispatch({
  name: 'SEND_EMAIL',
  target: 'EMAIL_SERVICE',
  data: {
    to: 'example@email.com',
    message: 'Hello World!',
  },
});
```

The receiving service will receive an object that looks like the dispatched action, plus a `source` field signifying the originating service.

``` js
{
  name: 'SEND_EMAIL',
  source: 'ORDER_SERVICE',
  target: 'EMAIL_SERVICE',
  data: {
    to: 'example@email.com',
    message: 'Hello World!',
  },
}
```

### Events

An event is a message that signifies an occurrence. An event is broadcast to all services that have subscribed to that event.
The response object can be used to check how many services have received (and acknowledged) the event.

``` js
import { EventDispatcher } from '@agnos-cloud/core';

const dispatcher = new EventDispatcher();
const response = await dispatcher.dispatch({
  name: 'EMAIL_SENT',
  data: {
    id: 123,
    to: 'example@email.com',
    message: 'Hello World!',
  },
});

// how many services have received the event
const received = await response.getReceivedServices().length;

// how many services have acknowledged the event
const received = await response.getAcknowledgedServices().length;
```

Agnos will dispatch the event to all appropriate services; if no service is found no error is thrown or returned.

It should be possible to specify the name of a service to which the event must be sent.

``` js
import { EventDispatcher } from '@agnos-cloud/core';

const dispatcher = new EventDispatcher();
const response = await dispatcher.dispatch({
  name: 'EMAIL_SENT',
  target: 'EMAIL_SERVICE',
  data: {
    id: 123,
    to: 'example@email.com',
    message: 'Hello World!',
  },
});
```

The receiving service will receive an object that looks like the dispatched event, plus a `source` field signifying the originating service.

``` js
{
  name: 'EMAIL_SENT',
  source: 'ORDER_SERVICE',
  target: 'EMAIL_SERVICE',
  data: {
    id: 123,
    to: 'example@email.com',
    message: 'Hello World!',
  },
}
```

## Health Checks

A component should expose a health check endpoint.

A health check system should periodically ping each service which, in turn, should ping its components.
The result of the check should be stored and accessible via a dashboard. Additionally, failing checks
should result in retries/restarts and/or alerts/alarms.

## Rerences

- https://kubernetes.io/docs/home/
  - [https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/
