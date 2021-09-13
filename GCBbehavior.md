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





## Other Notes


Line 239 index.go- we only respect the explicitly set status code if there is no message. There are instances where there is a message, but we aren't already failing (memory limit exceeded but timeout not expired). In that situation, we will pass a healthcheck despite explicitly setting to fail




Line 198 index.go- switch statement can be eliminated. It will always only run the default path, as it checks if the instancecount is > 1 before the switch statement