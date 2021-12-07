---
title: 'Metrics'
excerpt: 'This section covers the important aspect of metrics management in k6. How and what kind of metrics k6 collects automatically (_built-in_ metrics), and what custom metrics you can make k6 collect.'
---

This section covers the important aspect of metrics management in k6. How and what kind of metrics k6 collects automatically (_built-in_ metrics), and what custom metrics you can make k6 collect.

## Built-in metrics

The _built-in_ metrics are the ones you can see output to stdout when you run the simplest possible k6 test:

<CodeGroup lineNumbers={[true]}>

```javascript
import http from 'k6/http';

export default function () {
  http.get('https://test-api.k6.io/');
}
```

</CodeGroup>

Running the above script will output something like below:

<CodeGroup labels={["output"]} lineNumbers={[false]}>

```bash
$ k6 run script.js

          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: http_get.js
     output: -

  scenarios: (100.00%) 1 scenario, 1 max VUs, 10m30s max duration (incl. graceful stop):
           * default: 1 iterations for each of 1 VUs (maxDuration: 10m0s, gracefulStop: 30s)


running (00m03.8s), 0/1 VUs, 1 complete and 0 interrupted iterations
default ✓ [======================================] 1 VUs  00m03.8s/10m0s  1/1 iters, 1 per VU

     data_received..................: 22 kB 5.7 kB/s
     data_sent......................: 742 B 198 B/s
     http_req_blocked...............: avg=1.05s    min=1.05s    med=1.05s    max=1.05s    p(90)=1.05s    p(95)=1.05s
     http_req_connecting............: avg=334.26ms min=334.26ms med=334.26ms max=334.26ms p(90)=334.26ms p(95)=334.26ms
     http_req_duration..............: avg=2.7s     min=2.7s     med=2.7s     max=2.7s     p(90)=2.7s     p(95)=2.7s
       { expected_response:true }...: avg=2.7s     min=2.7s     med=2.7s     max=2.7s     p(90)=2.7s     p(95)=2.7s
     http_req_failed................: 0.00% ✓ 0        ✗ 1
     http_req_receiving.............: avg=112.41µs min=112.41µs med=112.41µs max=112.41µs p(90)=112.41µs p(95)=112.41µs
     http_req_sending...............: avg=294.48µs min=294.48µs med=294.48µs max=294.48µs p(90)=294.48µs p(95)=294.48µs
     http_req_tls_handshaking.......: avg=700.6ms  min=700.6ms  med=700.6ms  max=700.6ms  p(90)=700.6ms  p(95)=700.6ms
     http_req_waiting...............: avg=2.7s     min=2.7s     med=2.7s     max=2.7s     p(90)=2.7s     p(95)=2.7s
     http_reqs......................: 1     0.266167/s
     iteration_duration.............: avg=3.75s    min=3.75s    med=3.75s    max=3.75s    p(90)=3.75s    p(95)=3.75s
     iterations.....................: 1     0.266167/s
     vus............................: 1     min=1      max=1
     vus_max........................: 1     min=1      max=1
```

</CodeGroup>

All the `http_req_...` lines and the ones after them are _built-in_ metrics that get written to stdout at the end of a test.

The following _built-in_ metrics will **always** be collected by k6:

| Metric Name | Type | Description |
| ----------- | ---- | ----------- |
| vus                | Gauge   | Current number of active virtual users |
| vus_max            | Gauge   | Max possible number of virtual users (VU resources are pre-allocated, to ensure performance will not be affected when scaling up the load level) |
| iterations         | Counter | The aggregate number of times the VUs in the test have executed the JS script (the `default` function). |
| iteration_duration | Trend   | The time it took to complete one full iteration. It includes the time spent in `setup` and `teardown` as well. See the workaround in the example below, how to create a sub-metric for getting only the duration of the iteration's function for a specific scenario. |
| dropped_iterations | Counter | Introduced in k6 v0.27.0, the number of iterations that could not be started due to lack of VUs (for the arrival-rate executors) or lack of time (due to expired maxDuration in the iteration-based executors). |
| data_received      | Counter | The amount of received data. Read [this example](/examples/track-transmitted-data-per-url) to track data for an individual URL. |
| data_sent          | Counter | The amount of data sent. Read [this example](/examples/track-transmitted-data-per-url) to track data for an individual URL. |
| checks             | Rate    | The rate of successful checks. |

<Collapsible title="Workaround to calculate `iteration_duration` metric only for a scenario">

A common requested case is to track the `iteration_duration` metric without including time spent for `setup` and `teardown` functions.
This feature is not yet available but a threshold on `iteration_duration` can be used as a workaround.

It's based on the concept of creating a sub-metrics for the required scope and set a always-pass threshold. It mostly works with every filter that already works with the threshold, for example:
* 'iteration_duration{scenario:default}' will generate a sub-metric collecting samples only for the default scenario's iteration. 'scenario:default' because that's the internal k6 scenario name when we haven't actually specified `options.scenarios` explicitly and are just using the execution shortcuts instead.
* 'iteration_duration{group:::setup}' or 'iteration_duration{group:::teardown}' create sub-metrics colecting duration only for `setup` and `teardown`. '::' is the group separator syntax and k6 implicitly creates groups for `setup` and `teardown`.
* 'http_req_duration{scenario:default}' can be useful as well for isolating executed long-running requests.

<CodeGroup lineNumbers={[true]}>

```javascript
import { sleep } from 'k6'
import http from 'k6/http'

export const options = {
    vus: 1,
    iterations: 1,
    thresholds: {
        'iteration_duration{scenario:default}': [`max>=0`],
        'iteration_duration{group:::setup}': [`max>=0`],
        'iteration_duration{group:::teardown}': [`max>=0`],
        'http_req_duration{scenario:default}': [`max>=0`],
    },
}

export function setup() {
    http.get('http://httpbin.test.k6.io/delay/5');
}
export default function () {
    http.get('http://test.k6.io/?where=default');
    sleep(0.5);
}

export function teardown() {
    http.get('http://httpbin.test.k6.io/delay/3');
    sleep(5);
}
```

</CodeGroup>

Dedicated sub-metrics have been generated collecting samples only for the scope defined by thresholds:

<CodeGroup lineNumbers={[false]}>

```
     http_req_duration..............: avg=1.48s    min=101.95ms med=148.4ms  max=5.22s    p(90)=4.21s    p(95)=4.71s
       { expected_response:true }...: avg=1.48s    min=101.95ms med=148.4ms  max=5.22s    p(90)=4.21s    p(95)=4.71s
     ✓ { scenario:default }.........: avg=148.4ms  min=103.1ms  med=148.4ms  max=193.7ms  p(90)=184.64ms p(95)=189.17ms

     iteration_duration.............: avg=5.51s    min=1.61s    med=6.13s    max=8.81s    p(90)=8.27s    p(95)=8.54s
     ✓ { group:::setup }............: avg=6.13s    min=6.13s    med=6.13s    max=6.13s    p(90)=6.13s    p(95)=6.13s
     ✓ { group:::teardown }.........: avg=8.81s    min=8.81s    med=8.81s    max=8.81s    p(90)=8.81s    p(95)=8.81s
     ✓ { scenario:default }.........: avg=1.61s    min=1.61s    med=1.61s    max=1.61s    p(90)=1.61s    p(95)=1.61s
```

</CodeGroup>

</Collapsible>

## HTTP-specific built-in metrics

_built-in_ metrics will only be generated when/if HTTP requests are made:

| Metric Name                          | Type    | Description                                                                                                                                                                                                                                  |
| ------------------------------------ | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| http_reqs                            | Counter | How many HTTP requests has k6 generated, in total.                                                                                                                                                                                           |
| http_req_blocked                     | Trend   | Time spent blocked (waiting for a free TCP connection slot) before initiating the request. `float`                                                                                                                                           |
| http_req_connecting                  | Trend   | Time spent establishing TCP connection to the remote host. `float`                                                                                                                                                                           |
| http_req_tls_handshaking             | Trend   | Time spent handshaking TLS session with remote host                                                                                                                                                                                          |
| http_req_sending                     | Trend   | Time spent sending data to the remote host. `float`                                                                                                                                                                                          |
| http_req_waiting                     | Trend   | Time spent waiting for response from remote host (a.k.a. \"time to first byte\", or \"TTFB\"). `float`                                                                                                                                       |
| http_req_receiving                   | Trend   | Time spent receiving response data from the remote host. `float`                                                                                                                                                                             |
| http_req_duration                    | Trend   | Total time for the request. It's equal to `http_req_sending + http_req_waiting + http_req_receiving` (i.e. how long did the remote server take to process the request and respond, without the initial DNS lookup/connection times). `float` |
| http_req_failed <sup>(≥ v0.31)</sup> | Rate    | The rate of failed requests according to [setResponseCallback](/javascript-api/k6-http/setresponsecallback-callback).                                                                                                                        |

### Accessing HTTP timings from a script

If you want to access the timing information from an individual HTTP request in the k6, the [Response.timings](/javascript-api/k6-http/response) object provides the time spent on the various phases in `ms`:

- blocked: equals to `http_req_blocked`.
- connecting: equals to `http_req_connecting`.
- tls_handshaking: equals to `http_req_tls_handshaking`.
- sending: equals to  `http_req_sending`.
- waiting: equals to `http_req_waiting`.
- receiving: equals to `http_req_receiving`.
- duration: equals to `http_req_duration`.

<CodeGroup lineNumbers={[true]}>

```javascript
import http from 'k6/http';

export default function () {
  const res = http.get('http://httpbin.org');
  console.log('Response time was ' + String(res.timings.duration) + ' ms');
}
```

</CodeGroup>

Below is the expected (partial) output:

<CodeGroup lineNumbers={[false]}>

```bash
$ k6 run script.js

  INFO[0001] Response time was 337.962473 ms               source=console
```

</CodeGroup>

## Custom metrics

You can also create your own metrics, that are reported at the end of a load test, just like HTTP timings:

<CodeGroup lineNumbers={[true]}>

```javascript
import http from 'k6/http';
import { Trend } from 'k6/metrics';

const myTrend = new Trend('waiting_time');

export default function () {
  const r = http.get('https://httpbin.org');
  myTrend.add(r.timings.waiting);
  console.log(myTrend.name); // waiting_time
}
```

</CodeGroup>

The above code will create a Trend metric named “waiting_time” and referred to in the code using the variable name `myTrend`.

Custom metrics will be reported at the end of a test. Here is how the output might look:

<CodeGroup lineNumbers={[false]}>

```bash
$ k6 run script.js

  ...
  INFO[0001] waiting_time                                  source=console

  ...
  iteration_duration.............: avg=1.15s    min=1.15s    med=1.15s    max=1.15s    p(90)=1.15s    p(95)=1.15s   
  iterations.....................: 1     0.864973/s
  waiting_time...................: avg=265.245396 min=265.245396 med=265.245396 max=265.245396 p(90)=265.245396 p(95)=265.245396
```

</CodeGroup>

## Metric types

All metrics (both the _built-in_ ones and the custom ones) have a type. The four different metric types in k6 are:

| Metric type                                   | Description                                                                                              |
| --------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| [Counter](/javascript-api/k6-metrics/counter) | A metric that cumulatively sums added values.                                                            |
| [Gauge](/javascript-api/k6-metrics/gauge)     | A metric that stores the min, max and last values added to it.                                           |
| [Rate](/javascript-api/k6-metrics/rate)       | A metric that tracks the percentage of added values that are non-zero.                                   |
| [Trend](/javascript-api/k6-metrics/trend)     | A metric that allows for calculating statistics on the added values (min, max, average and percentiles). |

All values added to a custom metric can optionally be [tagged](/using-k6/tags-and-groups) which can be useful when analysing the test results.

### Counter _(cumulative metric)_

<CodeGroup lineNumbers={[true]}>

```javascript
import { Counter } from 'k6/metrics';

const myCounter = new Counter('my_counter');

export default function () {
  myCounter.add(1);
  myCounter.add(2);
}
```

</CodeGroup>

The above code will generate the following output:

<CodeGroup lineNumbers={[false]}>

```bash
$ k6 run script.js

  ...
  iteration_duration...: avg=16.48µs min=16.48µs med=16.48µs max=16.48µs p(90)=16.48µs p(95)=16.48µs
  iterations...........: 1   1327.67919/s
  my_counter...........: 3   3983.037571/s
```

</CodeGroup>

The value of `my_counter` will be 3 (if you run it one single iteration - i.e. without specifying --iterations or --duration).

Note that there is currently no way of accessing the value of any custom metric from within JavaScript. Note also that counters that have value zero (`0`) at the end of a test are a special case - they will **NOT** be printed to the stdout summary.

### Gauge _(keep the latest value only)_

<CodeGroup lineNumbers={[true]}>

```javascript
import { Gauge } from 'k6/metrics';

const myGauge = new Gauge('my_gauge');

export default function () {
  myGauge.add(3);
  myGauge.add(1);
  myGauge.add(2);
}
```

</CodeGroup>

The above code will result in an output like this:

<CodeGroup lineNumbers={[false]}>

```bash
$ k6 run script.js

  ...
  iteration_duration...: avg=21.74µs min=21.74µs med=21.74µs max=21.74µs p(90)=21.74µs p(95)=21.74µs
  iterations...........: 1   1293.475322/s
  my_gauge.............: 2   min=1         max=3
```

</CodeGroup>

The value of `my_gauge` will be 2 at the end of the test. As with the Counter metric above, a Gauge with value zero (`0`) will **NOT** be printed to the stdout summary at the end of the test.

### Trend _(collect trend statistics (min/max/avg/percentiles) for a series of values)_

<CodeGroup lineNumbers={[true]}>

```javascript
import { Trend } from 'k6/metrics';

const myTrend = new Trend('my_trend');

export default function () {
  myTrend.add(1);
  myTrend.add(2);
}
```

</CodeGroup>

The above code will make k6 print output like this:

<CodeGroup lineNumbers={[false]}>

```bash
$ k6 run script.js

  ...
  iteration_duration...: avg=20.78µs min=20.78µs med=20.78µs max=20.78µs p(90)=20.78µs p(95)=20.78µs
  iterations...........: 1   1217.544821/s
  my_trend.............: avg=1.5     min=1       med=1.5     max=2       p(90)=1.9     p(95)=1.95
```

</CodeGroup>

A trend metric is a container that holds a set of sample values, and which we can ask to output statistics (min, max, average, median or percentiles) about those samples.
By default, k6 will print average, min, max, median, 90th percentile, and 95th percentile.

### Rate _(keeps track of the percentage of values in a series that are non-zero)_

<CodeGroup lineNumbers={[true]}>

```javascript
import { Rate } from 'k6/metrics';

const myRate = new Rate('my_rate');

export default function () {
  myRate.add(true);
  myRate.add(false);
  myRate.add(1);
  myRate.add(0);
}
```

</CodeGroup>

The above code will make k6 print output like this:

<CodeGroup lineNumbers={[false]}>

```bash
$ k6 run script.js

  ...
  iteration_duration...: avg=22.12µs min=22.12µs med=22.12µs max=22.12µs p(90)=22.12µs p(95)=22.12µs
  iterations...........: 1      1384.362792/s
  my_rate..............: 50.00% ✓ 2           ✗ 2
```

</CodeGroup>

The value of `my_rate` at the end of the test will be 50%, indicating that half of the values
added to the metric were non-zero.

### Notes

- custom metrics are only collected from VU threads at the end of a VU iteration, which means that
  for long-running scripts, you may not see any custom metrics until a while into the test.

## Metric graphs in k6 Cloud Results

If you use [k6 Cloud Results](/cloud/analyzing-results/overview), you have access to all test
metrics within the [Analysis Tab](/cloud/analyzing-results/analysis-tab). You can use this tab to further analyze and
compare test result data, to look for meaningful correlations in your data.

![k6 Cloud Analysis Tab](images/Metrics/cloud-insights-analysis-tab.png)
