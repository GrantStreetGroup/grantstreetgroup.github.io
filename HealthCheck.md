---
title: Grant Street Group HealthCheck Standard
---
# {{ page.title }}

A definition of the result format for the results of a HealthCheck for an application.
Loosely based on the [FT Health Check Standard](https://github.com/Financial-Times/fettle/blob/master/FTHealthcheckstandard.pdf),
with much simplification and the major addition of the "results" field that allows nesting results.
These nested results provide much power to see a summary of the state and yet dig deeper into why something has failed.
Tag support means you can query for just the features you are interested in and not have to wait for other results to be calculated.

## Checker Implementations

*   [Perl HealthCheck Module](https://MetaCPAN.org/pod/HealthCheck)

Other useful code
-----------------

*   [HealthCheck::Diagnostic](https://MetaCPAN.org/pod/HealthCheck::Diagnostic) - Base class for writing HealthCheck Diagnostic Checks
    *   As well as [some documentation on writing one](https://metacpan.org/pod/distribution/HealthCheck/lib/HealthCheck/WritingADiagnostic.pod).
*   [Plack::Middleware::HealthCheck](https://metacpan.org/pod/Plack::Middleware::HealthCheck) - HealthChecks over the web
    *   Note this implements the correct HTTP response statuses
*   More [HealthCheck::Diagnostic checks on the MetaCPAN](https://MetaCPAN.org/search?q=HealthCheck%3A%3ADiagnostic)

### Checker Implementation Best Practices

The checker SHOULD summarize the “status” field, picking the worst severity of all responses.

A checker SHOULD return the list of tags it checked. An individual check MAY return all the tags it has associated.

A checker MUST timeout in a reasonable amount of time and return an appropriate response for not getting results. For example, some endpoints may be hit every ten seconds and expected response time is shorter than that.

For HTTP(s) health checks, the HTTP response code MUST be a 200 if the app is in good health, and MUST be 503 in bad health. See also [Google's standard](https://cloud.google.com/load-balancing/docs/health-check-concepts#criteria-protocol-http).

Results
-------

A health check checker implementation should return a data structure that can be round-tripped into JSON. Fields in the result are expected to comply with definitions as described here. Results may include fields not described here and these additional fields should be included if the result is stored or forwarded.

```
 {
   "id" : "my_app",
   "status" : "CRITICAL",
   "label" : "My App's Health Check",
   "info" : "Something has gone terribly wrong",
   "timestamp" : "2001-02-03 04:05:06Z",
   "runbook" : "https://grantstreetgroup.github.io/HealthCheck.html",
   "results" : [
      {
         "id" : "simple_check",
         "status" : "OK",
         "runtime" : 0.012
      },
      {
         "id" : "named_check",
         "status" : "OK",
         "label" : "Pretty Label",
         "runtime" : 0.027
      },
      {
         "id" : "timed_out",
         "status" : "UNKNOWN",
         "info" : "Timed out after too many seconds",
         "runtime" : 15.009
      },
      {
         "id" : "aggregate_check",
         "status" : "CRITICAL",
         "label" : "A check that aggregates different problems",
         "runbook" : "https://grantstreetgroup.github.io/HealthCheck.html",
         "results" : [
            {
               "id" : "subcheck_1",
               "status" : "OK"
            },
            {
               "id" : "subcheck_2",
               "status" : "OK",
               "label" : "Before the dawn of time",
               "timestamp" : "1969-12-31T12:59:59+00:00"
            },
            {
               "id" : "subcheck_3",
               "status" : "CRITICAL",
               "info" : "Check failed to do the thing it set out to do"
            }
         ],
         "runtime" : 2.087
      }
   ],
   "runtime" : 17.377,
   "tags" : [
      "my_app",
      "multi_check"
   ]
}
```

### Standard Result Fields

| Key | Constraints | Default | Notes |
|-----|-------------|---------|-------|
| id | Lowercase letters, numbers and underscores | index into the results list, if an aggregate, 0 otherwise. | Unique identifier for this check. Must be unique within in a group. |
| status | OK (0), WARNING (1), CRITICAL (2), UNKNOWN (3) | UNKNOWN | A string, the definition from the [Nagios Plugin Return Code Service State](https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/pluginapi.html). |
| label | String | "id" field | Displayed to a human describing the check. |
| info | String | "status" field | Displayed to a human describing the status. May include more detail than the status.  |
| timestamp | [RFC3339 timestamp](https://tools.ietf.org/html/rfc3339) | timestamp of a "parent" result or a top-level result will default to the current time.  | The top-level result should fill this out when it is run. Checks that do caching of results should use the timestamp of the last time the result was updated.  It is highly recommended that you stick with GMT, or the "Z"/+00:00 timezone for consistency.  RFC3339 is a subset of [ISO8601](https://en.wikipedia.org/wiki/ISO_8601). |
| results | List of additional check results | None | If this is an aggregate check that combines sub-checks, this can be a list of the results for each sub-check. Each result must comply with the constraints defined here. |
| runtime | Float (any precision) | None; may be auto-calculated at top-level | The amount of time it took to run the check, in seconds. Individual results can have their own individual runtimes. The checks SHOULD NOT use cached values, and only reflect the real amount of time it took to run. Checks could be ran in parallel, so summarizing runtimes for the parent check may not be appropriate, and parent checks should use their own timers. |
| runbook | A troubleshooting runbook link | None | A string of an URL that is linked to a troubleshooting runbook when healthcheck is not in OK status. |
| tags | Array of strings | None | A set of tags that can be used to classify the result. These can generally be filtered with a separate "tags" query from the check implementation.  |
| data | A freeform structure | None | A freeform machine-readable set of data, providing additional details to the test. The structure and keys SHOULD be consistent if the same type of test is run multiple times.  |

Default values are what the result reader is expected to assume, not necessarily provided by the check implementation.

### Proposed Result Fields

These are fields that are thought might be useful as standard fields, but in order to keep things as simple as possible we are not putting them there until we find out if they are actually used. Please add any custom fields you may add to your implementation here, along with what code is using it.

| Used | Key | Constraints | Default | Notes |
|------|-----|-------------|---------|-------|
| None | severity | Known severity level | None | How important is this check. Expected to be a syslog severity level. How the result reader uses this information is implementation specific. |
| None | facility | String | None | The type of check. Also from syslog, |
| None | businessImpact | URI | None | From the [FT Health Check Standard](https://github.com/Financial-Times/fettle/blob/master/FTHealthcheckstandard.pdf), a link that a business person can follow to understand what the implications of this failing are. The "runbook" is likely to replace this field. |
| None | technicalSummary | URI | None | From the [FT Health Check Standard](https://github.com/Financial-Times/fettle/blob/master/FTHealthcheckstandard.pdf), a link describing what this is checking on a technical level. The "runbook" is likely to replace this field. |
| None | panicGuide | URI | None | From the [FT Health Check Standard](https://github.com/Financial-Times/fettle/blob/master/FTHealthcheckstandard.pdf), a link to instructions of what the prod supporter can do to resolve the failure. The "runbook" is likely to replace this field. |
