# Agnos Cloud

A cloud-agnostic platform for designing, deploying, and managing micro-services.

Something like https://railway.com/

![Screenshot 2024-12-20 at 23 20 37](https://github.com/user-attachments/assets/8dedc60d-5294-4f3e-bc5c-e14a0ca1df72)


## Key Concepts

### Service

A service is a logical representation of a microservice.

A service is a wrapper around resources (typically servers) that do the actual work of handling requests coming to the service.

A service has a name with which it can always be referenced regardless of where its resources are located. _Make rules for legitimate names._
The service name must be unique within the _application_.

A service can have input variables.

A service can have output variables, including the `url` output which defaults to _<SERVICE NAME>_URL_.

A service can expose an interface. An interface is the set of actions a service exposes as well as the
schema for the expected inputs for those actions.

A service can have persistent storage so that data written to such storage persists across restarts.

``` yaml
API: 0.1
version: 1

services:
- name: email-service
  version: 2
  extends: email-service@1
  template: @agnos/email-service
  dependsOn:
  - service-a
  - service-b
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
  inputs:
    from_email_template: string
    service_a_url:
      type: string
      value:
        from: ${service-a.url}
  outputs:
    url: string
    smtp_server: string
  metrics:
    emails_sent_per_month:
      type: number
      update_frequency: monthly
```



### Resource

A resource is an executable package.

A resource  can be created from an HTTP server, a Docker image, a config file, a Git repo, or a local folder.

A resource runs in a service and can be accessed via that service.

A resource can save data to the persistent storage of its service.

``` yaml
API: 0.1
version: 1

resources:
- name: sendgrid
  version: v1
  source: https://hub.docker.com/r/sendgrid/sendgrid-python/
  sourceType: docker-image
  healthCheck: /healthcheck
- name: sendgrid-server
  version: v1
  source: https://my-sendgrid-server.com
  sourceType: http-server
  healthCheck: /health
```

### Deployment

A deployment creates a runtime environment, deploys a resource to that environment, and makes the resource accessible via its service.

``` yaml
API: 0.1
version: 1

environments:
- test
- local
- staging
- prod

deployments:
- name: deploy-email-local
  environment: local
  service: email-service@2
  resource: sendgrid@v1
  platform:
    name: docker-compose
    mem: 256
    cpu: 512
  ports:
  - container: 3000
    host: 3000
  scale:
    min: 1
    max: 3
    trigger:
      cpu: "70%"
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
  env:
    A: "a"
    B: ${secret.B}
- name: deploy-email-prod
  environment: prod
  service: email-service@2
  resource: sendgrid-server@v1
```

It should be possible to have a flow chat of services showing their inter-dependencies, similar to what is shown below.

![Screenshot 2024-11-22 at 13 03 01](https://github.com/user-attachments/assets/50feb2b6-3b26-4432-9fab-3dfb0cb7b07f)

![Screenshot 2024-11-22 at 13 06 31](https://github.com/user-attachments/assets/3b7300c2-f624-4d8e-872b-5c402a7b3452)

### Deployment Platform

Developers should be able to create custom deployment platforms.

To create a deployment platform, register an HTTP server we can call at various lifecycles of the deployment.

### Referencing a Service

A service, ServiceA, can reference another service, ServiceB (as long as there's no cyclical dependence).
During deployment, ServiceB will be deployed before ServiceA, and the URL of ServiceB will be an environment
variable for ServiceA.

You can also reference a service directly using the SDK + auto-generated code.


``` js
// import services from '@agnos-cloud/services';

// const serviceB = await services.getUrl('ServiceB');

// =========

import agnos from '...auto-generated-folder';

// to get a service
const emailService = agnos.services.emailService;

// to get an output variable
console.log(emailService.url);
```

### Actions

An action is a message sent from one service to another. It is synchronous, meaning the response to the action is expected
in the same request-response cycle.

Code that _maps_ service actions to resource endpoints is downloaded during code generation.

``` js
/*
import { ActionDispatcher } from '@agnos-cloud/core';

const dispatcher = new ActionDispatcher();
const response = await dispatcher.dispatch({
  name: 'SEND_EMAIL',
  data: {
    to: 'example@email.com',
    message: 'Hello World!',
  },
});
*/

import agnos from '...auto-generated-folder';

const emailService = agnos.services.emailService;
await emailService.sendEmail({
    to: 'example@email.com',
    message: 'Hello World!',
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

## Marketplace

There will be a marketplace where developers can share services, resources, and deployments.

By reusing items from the marketplace we will be able to build robust applications in record time.

## Rerences

- https://kubernetes.io/docs/home/
  - https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
- https://developer.hashicorp.com
  - https://developer.hashicorp.com/terraform/language/stacks/use-cases
- https://developer.hashicorp.com/terraform/tutorials/configuration-language
- https://developer.hashicorp.com/terraform/tutorials/aws-get-started
