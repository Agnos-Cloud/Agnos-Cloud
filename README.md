# Agnos Cloud

A cloud-agnostic platform for designing, deploying, and managing micro-services.

## Key Concepts

### Service

A service is a logical representation of a microservice in a manner that does not change based on the host machine or environment.
In other words, the way you programmatically interact with a service should remain the same even across different deployment environments or
cloud service providers.

The actual work in a service is done by a component.

A service can have different components for different environments.

A service can have different environment variables for different environments.

A service has a name with which it can always be referenced.

A service can have inputs.

A service can have outputs, including the URL.

A service can expose an interface.

### Component

A component is an executable package.

A component  can be created from an HTTP server, a Docker image, a Git repo, or a local folder.

A component can implement an interface

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

## Rerences

- https://kubernetes.io/docs/home/
  - https://kubernetes.io/docs/concepts/overview/#why-you-need-kubernetes-and-what-can-it-do
