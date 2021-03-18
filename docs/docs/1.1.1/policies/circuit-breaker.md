# Circuit Breaker

This policy will look for errors in the live traffic being exchanged between our data plane proxies and it will mark a data proxy as an unhealthy if certain conditions are met and - by doing so - making sure that no additional traffic can reach an unhealthy data plane proxy until it is healthy again.

Circuit breakers - unlike active [Health Checks](/docs/1.1.1/policies/health-check/) - do not send additional traffic to our data plane proxies but they rather inspect the existing service traffic. They are also commonly used to prevent cascading failures in our services.

:::tip
Like a real-world circuit breaker when the circuit is **closed** then traffic between a source and destination data plane proxy is allowed to freely flow through it, and when it is **open** then the traffic is interrupted.
:::

The conditions that determine when a circuit breaker is closed or open are being configured in what we call "detectors". This policy provides 5 different types of detectors and they are triggered on some deviations in the upstream service behavior. All detectors could coexist on the same outbound interface.

Once one of the detectors has been triggered the corresponding data plane proxy is ejected from the set of the load balancer for a period equal to [baseEjectionTime](#baseejectiontime). Every further ejection of the same data plane proxy will further extend the [baseEjectionTime](#baseejectiontime) multiplied by the number of ejections: for example the 4th ejection will be lasting for a period of time of `4 * baseEjectionTime`.

Available detectors:
- [Total Errors](#total-errors)
- [Gateway Errors](#gateway-errors)
- [Local Errors](#local-errors)
- [Standard Deviation](#standard-deviation)
- [Failures](#failures)

## Usage

As usual, we can apply `sources` and `destinations` selectors to determine how circuit breakers will be applied across our data plane proxies.

For example:

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
```yaml
apiVersion: kuma.io/v1alpha1
kind: CircuitBreaker
mesh: default
metadata:
  name: circuit-breaker-example
spec:
  sources:
  - match:
      kuma.io/service: web
  destinations:
  - match:
      kuma.io/service: backend
  conf:
    interval: 5s
    baseEjectionTime: 30s
    maxEjectionPercent: 20
    splitExternalAndLocalErrors: false 
    detectors:
      totalErrors: 
        consecutive: 20
      gatewayErrors: 
        consecutive: 10
      localErrors: 
        consecutive: 7
      standardDeviation:
        requestVolume: 10
        minimumHosts: 5
        factor: 1.9
      failure:
        requestVolume: 10
        minimumHosts: 5
        threshold: 85
```
We will apply the configuration with `kubectl apply -f [..]`.
:::
::: tab "Universal"
```yaml
type: CircuitBreaker
mesh: default
name: circuit-breaker-example
sources:
- match:
    kuma.io/service: web
destinations:
- match:
    kuma.io/service: backend
conf:
  interval: 1s
  baseEjectionTime: 30s
  maxEjectionPercent: 20
  splitExternalAndLocalErrors: false 
  detectors:
    totalErrors: 
      consecutive: 20
    gatewayErrors: 
      consecutive: 10
    localErrors: 
      consecutive: 7
    standardDeviation:
      requestVolume: 10
      minimumHosts: 5
      factor: 1.9
    failure:
      requestVolume: 10
      minimumHosts: 5
      threshold: 85
```
We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/1.1.1/documentation/http-api).
:::
::::

The example demonstrates a complete configuration. A `CircuitBreaker` can also be configured in a simpler way by leveraging the default values of Envoy for any property that is not explicitly defined, for instance:

:::: tabs :options="{ useUrlFragment: false }"
::: tab "Kubernetes"
```yaml
apiVersion: kuma.io/v1alpha1
kind: CircuitBreaker
mesh: default
metadata:
  name: circuit-breaker-example
spec:
  sources:
  - match:
      kuma.io/service: web
  destinations:
  - match:
      kuma.io/service: backend
  conf:
    detectors:
      totalErrors: {}
      standardDeviation: {}
```
We will apply the configuration with `kubectl apply -f [..]`.
:::
::: tab "Universal"
```yaml
type: CircuitBreaker
mesh: default
name: circuit-breaker-example
sources:
- match:
    kuma.io/service: web
destinations:
- match:
    kuma.io/service: backend
conf:
  detectors:
    totalErrors: {}
    standardDeviation: {}
```
We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/1.1.1/documentation/http-api).
:::
::::

In this example when we get five errors in a row of any type (`5` is default Envoy value for `totalErrors.consecutive`) the data plane proxy will be ejected for `30s` the first time, `60s` for the second time, and so on.

::: warning
In the current version of Kuma `destinations` only supports the `service` tag.
:::

### interval
Time interval between ejection analysis sweeps. Defaults to 10s.

### baseEjectionTime
The base time that a data plane proxy is ejected for. The real time is equal to the base time multiplied by the number of times the data plane proxy has been ejected. Defaults to 30s.

### maxEjectionPercent
The maximum percent of an upstream Envoy cluster that can be ejected due to outlier detection. Defaults to 10% but will eject at least one data plane proxy regardless of the value.

### splitExternalAndLocalErrors

Activates Split Mode.

:::tip
**Split Mode:** There are two types of errors that might occur in a circuit breaker:

* Locally originated: errors triggered locally when estabilishing a connection at the TCP layer (ie: connection refused, connection reset).
* Externally originated: errors triggered remotely like a 5xx error in the response.

If Split Mode is off, Kuma won't distinguish errors by their origin and they will be counted together. If Split Mode is on, different parameters can be used to fine tune the detectors. All detectors counts errors according to the state of this parameter. 
:::

## Detectors

Below is a list of available detectors that can be configured in Kuma.

### Total Errors

Errors with status code 5xx and locally originated errors, in Split Mode just errors with status code 5xx. 

- `consecutive` - how many consecutive errors in a row will trigger the detector. Defaults to `5`.

### Gateway Errors

Subset of [totalErrors](#total-errors) related to gateway errors (502, 503 or 504 status code).

- `consecutive` - how many consecutive errors in a row will trigger the detector. Defaults to `5`.

### Local Errors

Taken into account only in Split Mode, number of locally originated errors.

- `consecutive` - how many consecutive errors in a row will trigger the detector. Defaults to `5`.

### Standard Deviation

Detection based on success rate, aggregated from every data plane proxy in the Envoy cluster.

- `requestVolume` - ignore data plane proxies with a number of requests less than `requestVolume`. Defaults to `100`.
- `minimumHosts` - ignore counting the success rate for an Envoy cluster if the number of data plane proxies with required `requestVolume` is less than `minimumHosts`. Defaults to `5`.
- `factor` - resulting threshold equals to `mean - (stdev * factor)`. Defaults to `1.9`.

### Failures

Detection based on success rate with an explicit threshold (unlike [standardDeviation](#standard-deviation)).

- `requestVolume` - ignore data plane proxies with a number of requests less than `requestVolume`. Defaults to `50`.
- `minimumHosts` - ignore counting the success rate for an Envoy cluster if the number of data plane proxies with required `requestVolume` is less than `minimumHosts`. Defaults to `5`.
- `threshold` - eject the data plane proxy if its percentage of failures is greater than - or equal to - this value. Defaults to `85`.