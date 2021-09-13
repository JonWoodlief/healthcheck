# Healthcheck Failure criteria

|                              | Informational Healthcheck | Liveness Probes       | Readiness Probes       | Global Load Balancer Healthcheck |
| ---------------------------- | ------------------------- | --------------------- | ---------------------- | -------------------------------- |
| **Path**                     | /healthcheck              | /healthcheck/liveness | /healthcheck/readiness | /healthcheck/globalreadiness     |
| Get Buildtag                 | Fail                      | Fail                  | Fail                   | Fail                             |
| DBHealthCheck                | Fail                      | Fail                  | Fail                   | Fail                             |
| Memory*                      |                           | Fail*                 |                        |                                  |
| Can Reach CM                 |                           |                       |                        | Fail                             |
| Explicitly set status code** | Fail                      |                       | Fail                   | Fail                             |

*Memory check checks the current memory usage of the istance. If it's above the configured limit, it'll set a random timeout between 10 seconds and two minutes. If memory checks continue to go over the limit on subsequent calls until after the timeout expires, the healthcheck will throw another warning and then start to fail

**status codes for healthchecks can be set by a global admin doing a PUT on /healthcheck.