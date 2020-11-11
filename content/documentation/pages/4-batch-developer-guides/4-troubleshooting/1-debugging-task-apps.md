---
path: 'batch-developer-guides/troubleshooting/debugging-task-apps/'
title: 'Debugging Batch applications'
description: 'Debugging Batch applications'
---

# Debugging Batch Applications

Applications should be runnable as a standard Java JAR, invoked via `java -jar`.
All dependencies such as databases and messaging middleware should be available and tested manually for connection issues.
Running and debugging applications is independent of deploying via SCDF and running on a platform.

## Project Build Errors

```mermaid
graph TD;
    A(Errors in IDE) --> B(Run using Maven CLI);
    B --> C{Success?}
    C -->|Yes| D(Fix IDE errors)
    C -->|No| E(Fix code errors)
```

Fixing errors in the IDE will be dependent on which IDE is used.
By verifying the build is successful outside the IDE, this helps rule out project specific issues.

## Application Startup Exceptions

```mermaid
graph TD;
    A(Application Exceptions) --> B(Review stacktrace)
    B --> C{General Coding Errors}
    C --> D(Fix Code)
    B --> E{Connection Exceptions}
    E --> |Yes| F{Valid configuration?}
    F --> |Yes| G(Test manually & apply any changes)
    F --> |No| H(Fix settings)
```

When the application starts up, exceptions may occur due to general coding errors or connectivity issues.
Review the application log file to find the exceptions and fix as appropriate.
Ensure all external services are reachable, correct credentials are provided and any other required information.

## General Application Debugging

```mermaid
graph TD;
    A(Review stacktrace) --> B(Enable JDWP for debugging);
    A(Review stacktrace) --> C(Enable Spring Integration debug logging);
```

Applications can be debugged as normal java processes as well as by utilizing verbose log output via the DEBUG logging level.

To debug your application using a debugger, add the following JVM argument when starting your application, for example:

`-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005`

By adding this parameter, your application will wait for a debugger to connect on port `5005` to start debugging.
This can be any port you choose that is not already in use.

Enabling DEBUG level logging can also be a useful aid.
There are two packages of interest Spring Batch (`org.springframework.batch`) and Spring Cloud Task (`logging.level.org.springframework.cloud.task`).
To enable debug logging of this batch package, add the following JVM argument when starting your application, for example:

`-Dlogging.level.org.springframework.batch=debug`
To enable debug logging of this task package, add the following JVM argument when starting your application, for example:

`-Dlogging.level.org.springframework.cloud.task=debug`

## Task Execution Not Recorded to Database

```mermaid
graph TD;
    A(Task Execution not recorded) --> B(Verify Database is Accessible)
    B --> C(Verify that database dependencies are present in pom.xml)
    C --> D{EnableTask Annotated?}
    D --> |Yes| E(Enable Task logging)
    D --> |No| F(Add EnableTask annotation)
    E --> G(Rerun Task and review logs)
```

Task applications record execution information to a database.
Ensure the proper class level annotation `@EnableTask` is present. Also verify that the proper database dependency is in your pom.xml or build.gradle.
And finally that the database connection properties are properly configured.

## Job Failed to Execute

```mermaid
graph TD;
    A(Job Failed To Execute) --> B(Verify Database is Accessible)
    B --> C(Verify that database dependencies are present)
    C --> D{EnableBatchProcessing Annotated?}
    D --> |Yes| E{Job already run successfully?}
    D --> |No| F(Add EnableBatchProcessing annotation)
    E --> |Yes| G(Rerun app with new parameters)
    E --> |NO| H(Enable Batch logging)
    H --> I(Rerun Batch and review logs)
```

Spring Batch requires that the application have access to a database and also that the `@EnableBatchProcessing` is present.

A common issue raised is that a user launches a batch app after it has already been successfully launched does not re-execute the `Job`. This is because batch apps by default are restartable. Thus if a batch `Job` fails during execution it can be restarted and pick up at the step where it failed on the next execution.
Each batch `Job` instance is unique. Thus if you want to start another `Job` instance you will need to update one of the `Job` parameters (command line arg).
Or when building the `Job` in your code establish a `Job` [incrementer](https://docs.spring.io/spring-batch/trunk/reference/html/configureJob.html#JobParametersIncrementer).

## Platforms

### Cloud Foundry

```mermaid
graph TD;
    A(Verify Manifest) --> B(Verify environment variables);
    B --> C(Services to bind created?)
    C --> |No| C
    C --> |Yes| D(Errors in application log files?)
    D --> |Yes| E(Resolve exceptions)
    D --> |No| F(Enable SI logging for appropriate channels)
```

If an application that runs locally, but fails when deployed to Cloud Foundry, first inspect the deployment manifest (manifest.yml) for correctness.
This includes any environment variables that must be set, services to bind to and those services created.
Inspect the application startup log for any exceptions to resolve.

### Kubernetes

```mermaid
graph TD;
    A(Verify Deployment files) --> B(Verify environment variables);
    B --> C{External services accessible?}
    C --> |No| C
    C --> |Yes| D{Pod in RUNNING state?}
    D --> |No| F(kubectl describe POD_NAME)
    F --> G{Errors in event table?}
    G --> |Yes| H(Resolve errors)
    G --> |No| I(kubectl logs POD_NAME)
    I --> |Yes| J{Application exceptions?}
    J --> |Yes| K(Resolve errors)
```

If an application that runs locally, but fails when deployed to Kubernetes, first inspect the deployment files for correctness.
This includes any environment variables that must be set, properties for services to connect to and those services available.
Describe the application pod's event table to see issues creating the pod such as image pull causes, health check failures, etc.
Inspect the application startup log for any exceptions to resolve.

#### Containers

```mermaid
graph TD;
    A(Create Container) --> B(Launch Container using 'docker run' with env variables and args);
    B --> C{Successful?}
    C --> |No| D(Update container creation) --> C1(Launch Container using 'docker run') --> C
    C --> |Yes| F(Create K8s pod spec yaml for container) -->
    G(Apply pod spec yaml in K8s instance) -->
    H{Successful?}
    H --> |Yes| I{Problem Resolved}
    H --> |No| J(Update Pod Spec Yaml) --> G
    I --> |Yes| K(Success)
    I --> |No| L(Open Git Issue for Data Flow)
```

As discussed before the application may run as expected from your local platform however, it still fails when launched from Spring Cloud Data Flow.
This could be because of how the container is created by the tool that is being used to create the container, for example: DockerFile, Spring Boot container Plugin, Jib, etc.
The chart above provides a guide on how to trouble shoot why apps may fail to launch because of container related issues.

[[note]]
| Data Flow uses the [Common application properties](https://docs.spring.io/spring-cloud-dataflow/docs/current/reference/htmlsingle/#_common_application_properties) and the [Spring Boot Common Tags](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-metrics-common-tags) to apply common metrics tags to all deployed `Task` applications. Those properties often use [Spring placeholders](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-placeholders-in-properties) to resolve their values, for example: `management.metrics.tags.application=${task.name:unknown}-${task.execution.id:unknown}`.
| Some image containers may not be able to parse this correctly and will replace the embedded properties with empty strings which may cause task apps to fail.
| If you are not using metrics you may disable these fields by setting the `spring.cloud.dataflow.applicationProperties.taskResource` to a non-existent file. For example: `spring.cloud.dataflow.applicationProperties.taskResource=classpath:fake.yml`
