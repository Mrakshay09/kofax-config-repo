# kofax-config-repo

Configuration repository consumed by the Kofax Capture **config-server**
(Spring Cloud Config Server) when it runs with the `git` profile.

## Layout

| File                            | Served to                                  |
| ------------------------------- | ------------------------------------------ |
| `application.yml`               | Every microservice (shared defaults)       |
| `<service-name>.yml`            | Only the matching service                  |
| `<service-name>-<profile>.yml`  | The service when that Spring profile is active (e.g. `-docker`, `-prod`) |

Two platform components do **not** fetch from config-server and have no file here:

- `config-server` (would be a chicken-and-egg)
- `discovery-service` (Eureka — bootstraps everything else)

## How config-server picks this up

`config-server/src/main/resources/application.yml` already defines a `git` profile:

```yaml
spring:
  config:
    activate:
      on-profile: git
  cloud:
    config:
      server:
        git:
          uri: ${CONFIG_GIT_URI:https://github.com/kofax/config-repo.git}
          default-label: ${CONFIG_GIT_BRANCH:main}
          username: ${CONFIG_GIT_USER:}
          password: ${CONFIG_GIT_PASS:}
          clone-on-start: true
          force-pull: true
```

To switch the local Docker stack from the bundled `native` profile to this git repo,
set the following on the `config-server` service in `docker-compose.yml`:

```yaml
environment:
  SPRING_PROFILES_ACTIVE: git
  CONFIG_GIT_URI: https://github.com/<your-org>/<this-repo>.git
  CONFIG_GIT_BRANCH: main
  # CONFIG_GIT_USER: <token-user>      # only for private repos
  # CONFIG_GIT_PASS: <token>
```

## Verifying

Once config-server is running, you can sanity-check what it serves:

```bash
curl -u configadmin:configadmin http://localhost:8888/api-gateway/docker
curl -u configadmin:configadmin http://localhost:8888/application/default
```

## Refresh without restarts

Services pick up new commits via the Spring Cloud Bus + `/actuator/refresh`
(or `/actuator/busrefresh` on config-server) — Kafka is already configured
as the bus transport.
