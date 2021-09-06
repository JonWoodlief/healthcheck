# Global Catalog HealthChecks



## Desired Flow

| Region       | **CM** | **GC** |
| ------------ | ------ | ------ |
| **US-South** | UP     | UP     |
| **EU-DE**    | UP     | UP     |
| **AU-SYD**   | UP     | UP     |

| Region       | **CM** | **GC** |
| :----------- | ------ | ------ |
| **US-South** | DOWN   | DOWN   |
| **EU-DE**    | UP     | UP     |
| **AU-SYD**   | UP     | UP     |

| Region       | **CM** | **GC** |
| ------------ | ------ | ------ |
| **US-South** | UP     | UP     |
| **EU-DE**    | DOWN   | DOWN   |
| **AU-SYD** | DOWN   | DOWN   |

| Region       | **CM** | **GC** |
| ------------ | ------ | ------ |
| **US-South** | DOWN   | UP     |
| **EU-DE**    | DOWN   | UP     |
| **AU-SYD** | DOWN   | UP     |

## 3 paths for healthchecks

1. `/healthcheck` - used by load balancer to determine if a region is healthy
   1. Failing `/healthcheck` removes an endpoint from the load balancer/origin pool
2. `/healthcheck/liveness` - used by Kubernetes to determine if pod is alive
   1. Failing `/healthcheck/liveness` results in the pod being killed and restarted
3. `/healthcheck/readiness` - used by Kubernetes to determine if pod can receive traffic
   1. Failing `/healthcheck/readiness` means services will no longer send traffic to that pod

These 3 paths all exist today in GC. All 3 routes point to the same function and return the same result (index.go:73)

Currently the Kubernetes readiness probe points to `/healthcheck` , and the load balancer healthchecks point to `/healthcheck/readiness`. These should be switched to be more explicit.

Failing the load balancer healthcheck is the only check we can fail, without resulting in a circular dependency

## When GC can't reach CM locally

In no situation do we fail `/healthcheck/liveness`- inability to reach CM should not result in restarts

1. Fail `/healthcheck/readiness`
   1. Kubernetes service will no longer route traffic to these pods
   2. When CM goes down globally, so does GC
      1. No easy way to turn GC back on when CM is down
   3. Introduces circular healthcheck dependency
      1. When CM is down, GC goes unready, can't be reached pod-to-pod
      2. Neither will come back up on their own
   4. **Option 1.5**- GC has available configuration to stop failing `/healthcheck/readiness` because it can't reach CM
      1. Can activate manually or via automation
      2. Where is the state of this configuration stored, and how is it set?
2. Don't fail any healthcheck
   1. Users will receive 503s
   2. Manually remove that region from rotation ASAP to avoid a CIE
      1. Have to manually add that region back when CM is back up in that region
      2. 15 minute timer for CIE
   3. Choose whether to create new alert, or rely on alert from CM going down?
3. Fail `/healthcheck`
   1. Remove a region from rotation when CM can't be reached locally
   2. doesn't block CM from coming up
   3. If CM goes down globally, GC goes down globally
   4. **Option 3.5** - failover load balancer
      1. when no region passes `/healthcheck`, fail over to second load balancer which targets `/healthcheck/readiness`
      2. If CM is down- we fail `/healthcheck` but pass `/healthcheck/readiness`, which means second load balancer will ignore CM's state
      3. If CM is down to one region, and GC in one region gets overwhelmed, GC will fail over to healthy regions
4. Only fail `/healthcheck` when MC's local endpoint can't be reached but it's global endpoint can be reached
   1. Highly available
   2. doesn't block CM from coming up
   3. If call to CM's global endpoint fails for reasons other than going down (egress failures), we won't respond to failures to reach local endpoint
   4. If GC's last region starts to go down, traffic won't be redistributed to other endpoints
5. GC switches to global endpoint
   1. Slow
   2. Compliance issues
   3. Highly available, doesn't block CM from coming up

## Conclusion

Never fail readiness probes or liveness probes when CM goes down locally, so that you don't interfere with CM's ability to come back up



Best options are-

1. Don't fail any healthchecks
   1. Users get 503s in regions where CM is down
   2. Always up and always globally load balance for vital GC functionality (regions)
   3. Devs on hook to take GC regions offline when CM has extended regional outage
2. Fail `/healthcheck` (used by load balancer)
   1. Remove a GC endpoint from global load balancing when CM is down
   2. Can fail over to secondary load balancer or origin pool when CM is down in all regions
3. GC switches to global endpoint
   1. Highly available and very simple
   2. **Significant** performance and compliance concerns
      1. Performance issues can be helped by only using global when local can't be reached
   3. Always up and always globally load balance for vital GC functionality (regions)

## Federated Healthcheck Changes

Add `"can_reach_cm"` check to federated healthcheck

```javascript
{

 "cm-api": {

  "healtcheck": {

   "globalcatalog-au-syd-prod-content-mgmt-ccb9cdf44-2nzct": 200,

   "globalcatalog-au-syd-prod-content-mgmt-ccb9cdf44-bqkkr": 200,

   "globalcatalog-au-syd-prod-content-mgmt-ccb9cdf44-tzbr4": 200

  },

  "server_build": {

   "globalcatalog-au-syd-prod-content-mgmt-ccb9cdf44-2nzct": "2021-09-02.6771.ypprod",

   "globalcatalog-au-syd-prod-content-mgmt-ccb9cdf44-bqkkr": "2021-09-02.6771.ypprod",

   "globalcatalog-au-syd-prod-content-mgmt-ccb9cdf44-tzbr4": "2021-09-02.6771.ypprod"

  },

  "status_code": 200,

  "url": "http://content-mgmt.cm-prod.svc.cluster.local/healthcheck"

 },

 "cm-ui": {

  "healtcheck": {

   "kub--globalcatalog-au-syd-prod--content-mgmt-ui-588585dfd-9s2vk": 200,

   "kub--globalcatalog-au-syd-prod--content-mgmt-ui-588585dfd-bpl49": 200,

   "kub--globalcatalog-au-syd-prod--content-mgmt-ui-588585dfd-k5jhv": 200

  },

  "server_build": {

   "kub--globalcatalog-au-syd-prod--content-mgmt-ui-588585dfd-9s2vk": "2021-08-30.4620.ypprod",

   "kub--globalcatalog-au-syd-prod--content-mgmt-ui-588585dfd-bpl49": "2021-08-30.4620.ypprod",

   "kub--globalcatalog-au-syd-prod--content-mgmt-ui-588585dfd-k5jhv": "2021-08-30.4620.ypprod"

  },

  "status_code": 200,

  "url": "http://content-mgmt-ui.cm-prod.svc.cluster.local/healthcheck"

 },

 "forwarded_for": [

  "104.54.73.104",

  "172.30.7.64"

 ],

 "gc": {

  "healtcheck": {

   "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-cwlrs": 200,

   "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-k6bn5": 200,

   "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-r6m7p": 200,

   "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-wdrzd": 200

  },
  “can_reach_cm”: {
    "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-cwlrs": 200,

   "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-k6bn5": 200,

   "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-r6m7p": 200,

   "kub--globalcatalog-au-syd-prod--resource-catalog-7cdb86766b-wdrzd": 200

  },

  "status_code": 200,

  "url": "http://resource-catalog.globalcatalog-prod.svc.cluster.local/healthcheck"

 },

 "host": "au-syd-cm-janitor.globalcatalog.cloud.ibm.com",

 "hostname": "content-mgmt-janitor-56f98cd549-brmxf",

 "instance_id": "globalcatalog-au-syd-prod-content-mgmt-janitor-56f98cd549-brmxf",

 "server_build": "2021-09-02.6771.ypprod",

 "server_version": "1.0.0",

 "service": "janitor-content-management",

 "status_code": 200

}
```
