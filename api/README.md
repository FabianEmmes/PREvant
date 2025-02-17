The image `aixigo/prevant` provides the REST-API in order to deploy containers and to compose them into reviewable application.

# Configuration

In order to configure PREvant create a [TOML](https://github.com/toml-lang/toml) file that is mounted to the container's path `/app/config.toml`.

## Container Options

Create a table `containers` with following options:

```toml
[containers]

# Restrict memory usage of containers
memory_limit = '1g'
```

## Issue Tracking options

Application names are compared to issues which will be linked to cards on the frontend. Therefore, the REST backend needs to be able to compare the application names with issue tracking information.

Currently, Jira as a tracking system is supported.

```toml
[jira]
host = 'https://jira.example.com'
user = ''
password = ''
```

## Services

PREvant provides central configuration options for services deployed through its REST-API. For example, you can define that PREvant mounts a secret for a specific service of an application.

### Secrets

As secrets in [Docker Swarm](https://docs.docker.com/engine/swarm/secrets/) and [Docker-Compose](https://docs.docker.com/compose/compose-file/#secrets) PREvant mounts secrets und `/run/secrets/<secret_name>`. Therefore, you can use following configuration section to define secrets for each service.

Following example provides two secrets for the service `nginx`, mounted at `/run/secrets/cert.pem` and `/run/secrets/key.pem`, available for the application `master`.

```toml
[services.nginx]
[[services.nginx.secrets]]
# The name of the secret mounted in the container.
name = "cert.pem"
# base64 encoded value of the secret
data = "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS…FUlRJRklDQVRFLS0tLS0K"
# An optional regular expression that checks if the secret has to be
# mounted for this service. Default is ".+" (any app)
appSelector = "master"

[[services.nginx.secrets]]
name = "key.pem"
data = "LS0tLS1CRUdJTiBFTkNSWVBURUQgUF…JVkFURSBLRVktLS0tLQo="
```

## Companions

It is possible to start containers that will be started when the client requests to create a new service. For example, if the application requires an [OpenID](https://en.wikipedia.org/wiki/OpenID_Connect) provider, it is possible to create a configuration that starts the provider for each application. Another use case might be a Kafka services that is required by the application.

Furthermore, it is also possible to create containers for each service. For example, for each service a database container could be started.

For these use cases following sections provide example configurations.

### Application Wide

If you want to include an OpenID provider for every application, you could use following configuration.

```toml
[companions.openid]
type = 'application'
image = 'private.example.com/library/openid:latest'
env = [ 'KEY=VALUE' ]
```

The provided values of `serviceName` and `env` can include the [handlebars syntax](https://handlebarsjs.com/) in order to access dynamic values.

Additionally, you could mount files that are generated from handlebars templates (example contains a properties generation):

```toml
[companions.openid.volumes]
"/path/to/volume.properties" = """
remote.services={{#each services~}}
  {{~#if (eq type 'instance')~}}
    {{name}}:{{port}},
  {{~/if~}}
{{~/each~}}
"""
```

Furthermore, you can provide labels through handlebars templating:

```toml
[companions.openid.labels]
"com.github.prevant" = "bar-{{application.name}}"
```

#### Template Variables

The list of available handlebars variables:

- `application`: The companion's application information
  - `name`: The application name
- `applicationPath`: The root path to the application.
- `services`: An array of the services of the application. Each element has following structure:
  - `name`: The service name which is equivalent to the network alias
  - `port`: The exposed port of the service
  - `type`: The type of service. For example, `instance`, `replica`, `app-companion`, or `service-companion`.

#### Handlebar Helpers

PREvant provides some handlebars helpers which can be used to generate more complex configuration files. See handlerbar's [block helper documentation](https://handlebarsjs.com/block_helpers.html) for more details.

- `{{#isCompanion <type>}}` A conditional handlerbars block helper that checks if the given service type matches any companion type.
- `isNotCompanion <type>` A conditional handlerbars block helper that checks if the given service type does not match any companion type.

### Service Based

The service-based companions works the in the same way as the application-based services. Make sure, that the `serviceName` is unique by using the handlebars templating.

```toml
[companions.service-name]
serviceName = '{{service.name}}-db'
image = 'postgres:11'
env = [ 'KEY=VALUE' ]

[companions.service-name.postgres.volumes]
"/path/to/volume.properties" == "…"
[companions.openid.labels]
"com.github.prevant" = "bar-{{application.name}}"
```


#### Template Variables

The list of available handlebars variables:

- `application`: The companion's application information
  - `name`: The application name
- `service`: The companion's service containing following fields:
  - `name`: The service name which is equivalent to the network alias
  - `port`: The exposed port of the service
  - `type`: The type of service. For example, `instance`, `replica`, `app-companion`, or `service-companion`.
