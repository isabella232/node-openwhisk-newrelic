<!--- when a new release happens, the VERSION and URL in the badge have to be manually updated because it's a private registry --->
[![npm version](https://img.shields.io/badge/%40nui%2Fopenwhisk--newrelic-1.2.0-blue.svg)](https://artifactory.corp.adobe.com/artifactory/npm-nui-release/@nui/openwhisk-newrelic/-/@nui/openwhisk-newrelic-1.2.0.tgz)

# node-openwhisk-newrelic

Library for gathering metrics from Apache OpenWhisk actions and sending them to New Relic

## NewRelic Insert API JSON format guidelines

Reference: <https://docs.newrelic.com/docs/insights/insights-data-sources/custom-data/send-custom-events-event-api>

* `eventType` required
* `timestamp` Unix epoch timestamp either in seconds or milliseconds
* Key-value pairs with `float` and `string` values only
* Limits:
  * 255 attributes
  * 255 characters in the attribute name
  * 4KB max length attribute value
  * 100,000 HTTP POST requests/min, 429 status after, counter reset of 1 minute window


## Usage

Initialize the New Relic Metrics agent. This will start a `setTimeout` that will send metrics if your action is close to timeout using `__OW_DEADLINE`
```
const NewRelic = require('@nui/openwhisk-newrelic');
const metrics = new NewRelic({
    newRelicEventsURL: 'https://insights-collector.newrelic.com/v1/accounts/<YOUR_ACOUNT_ID>/events',
    newRelicApiKey: 'YOUR_API_KEY',
});
```

Collect all your custom/background metrics in a separate object
```
const customMetrics = {
    data: "value",
    userName: "sampleUser"
}
```

Send your metrics to New Relic
```
await metrics.send('EVENT_TYPE', customMetrics);
```
You MUST call `activationFinished()` to stop the agent when you are done sending metrics, or when your action is finishing. This will clear the action timeout that began when the class instance was defined.
```
metrics.activationFinished();
```

### Action Timeout

The default behavior of the agent is it will begin a `setTimeout` that will send metrics right before the  action times out, using the `OW_DEADLINE` environment variable.

In case you want to opt out of the action timeout, (example: unit tests) there are two ways to opt out:

1. Pass in `disableActionTimeout` to options:
```
const metrics = new NewRelic({
    newRelicEventsURL: 'https://insights-collector.newrelic.com/v1/accounts/<YOUR_ACOUNT_ID>/events',
    newRelicApiKey: 'YOUR_API_KEY',
    disableActionTimeout: true
});
```
2. Set the environment variable: `DISABLE_ACTION_TIMEOUT_METRIC` to `true`:
`export DISABLE_ACTION_TIMEOUT_METRIC = true`

If either of these are set to true, there will be no action timeout and calling `activationFinished` is no longer necessary.


In case you want to pass custom metrics to the action timeout, you can define a callback function in New Relic options. The result of the callback will be added to the default metrics and sent at action timeout. If you do not define an `eventType`, it will default to `timeout`.:
```
const metrics = new NewRelic({
    newRelicEventsURL: 'https://insights-collector.newrelic.com/v1/accounts/<YOUR_ACOUNT_ID>/events',
    newRelicApiKey: 'YOUR_API_KEY',
    actionTimeoutMetricsCb: function () {
        return {
            eventType: 'error',
            ...customMetrics
        }
    }
});
```
