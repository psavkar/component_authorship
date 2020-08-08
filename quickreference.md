# Component API Reference

Pipedream components are Node.js modules that run on Pipedream's serverless infrastructure. This document is a reference for the component API. Please raise an issue or PR in this repo if you notice something out-of-date.

## Capabilities

- Trigger Node.js on HTTP requests, timers, cron schedules, or manually
- Accept user input on deploy via [CLI](https://docs.pipedream.com/cli/reference/#pd-deploy), [API](https://docs.pipedream.com/api/rest/#overview), or [UI](https://pipedream.com/sources)
- Connect to 300+ apps using Pipedream managed auth
- Use most npm pacakges with no `npm install` or `package.json` required
- Store and retrieve state using the [built-in key-value store](#servicedb)
- Emit data you process within the component, allowing you to trigger Pipedream hosted workflows or access it outside of Pipedream via API

## Getting Started

To help you get started, we created a step-by-step walkthrough that demonstrates how to:

- Deploy components to Pipedream using the CLI
- Invoke a component manually, or on a schedule or HTTP request
- Maintain state across component invocations
- Emit deduped events using the `unique` and `greatest` strategies
- Use Pipedream managed OAuth for an app
- Use npm packages in components

## Learn and Contribute

You can browse, instantiate and contribute to curated components in Pipedream's Github repo, or you can author your own and maintain your code via your standard CI/CD process.


# Contents

- [Component Structure](#component-structure)
- [Props](#props)
  * [User Input Props](#user-input-props)
  * [Interface Props](#interface-props)
    + [Timer](#timer)
    + [HTTP](#http)
  * [Service Props](#service-props)
    + [DB](#db)
  * [App Props](#app-props)
- [Hooks](#hooks)
- [Dedupe Strategies](#dedupe-strategies)
- [Emitting Events](#emitting-events)
- [Using npm packages](#using-npm-packages)

# Component Structure

Pipedream components export an object with the following properties:

```javascript
module.exports = {
  name: "",
  version: "",
  description: "",
  props: {},
  methods: {},
  hooks: {
    async activate() {},
    async deactivate() {},
  },
  dedupe: "",
  async run(event) {
    this.$emit()
  },
}
```

| Property        | Type    | Required?    | Description                                                                |
|-----------------|---------|-------------|----------------------------------------------------------------------------|
| `name` | `string` | required | The name of the component, a string which identifies components deployed to users' accounts. This name will show up in the Pipedream UI, in CLI output (for example, from pd list commands), etc. It will also be converted to a unique slug on deploy to refernce a specific component instance (it will be auto-incremented if not unique within a user account). |
| `version` | `string` | required | The component version. There are no constraints on the version, but consider using [semantic versioning](https://semver.org/). |
| `description` | `string` | recommended | The description will appear in the Pipedream UI to aid in discovery and to contextualize instantiated components |
| `props` | `object` | optional | Props are custom attributes you can register on a component. When a value is passed to a prop attribute, it becomes a property on that component instance. You can reference these properties in component code using `this` (e.g., `this.propName`). |
| `methods` | `object` | optional | Define component methods for the component instance. They can be referenced via `this` (e.g., `this.methodName()`). |
| `hooks` | `object` | optional | Hooks are functions that are executed when specific component lifecycle events occur. Currently supported hooks are `activate()` and `deactivate()` (they execute when the component is activated or deactivated). |
| `dedupe` | `string` | optional | You may specify a dedupe strategy (`unique`, `greatest`, `last`) to be applied to emitted events |
| `run` | `method` | required | Each time a component is invoked (for example, via HTTP request), its `run` method is called. The event that triggered the component is passed to run, so that you can access it within the method. Events are emitted using `this.$emit()`. |

# Props

 Props are custom attributes you can register on a component. When a value is passed to a prop attribute, it becomes a property on that component instance. You can reference these properties in component code using `this` (e.g., `this.propName`). 

| Prop Type        | Description    | 
|-------------|----------------|
| _User Input_ | User input props allow components to accept input on deploy |
| _Interface_  | Attaches a Pipedream interface to your component (e.g., an HTTP interface or timer) |
| _Service_  | Attaches a Pipedream service to  your component (e.g., a key-value database to maintain state) | 
| _App_ | Enables managed auth for a component |


## User Input Props

User input props allow components to accept input on deploy. When deploying a component, users will be prompted to enter values for these props, setting the behavior of the component accordingly.

```javascript
props: {
  myPropName: {
    type: "",
    label: "",
    description: "",
    options: [], // OR async options() {} to return dynamic options
    optional: true || false,
    propDefinition: [],
    default: 
  },
},
```

| Property        | Type    | Required? | Description |
|-------------|----------------|---------------|--------|
| `type`        | `string` | required | Value must be set to a valid prop type: `string` `string[]` `number` `boolean` |
| `label`        | `string` | optional | A friendly label to show to user for this prop. If a label is not provided, the `propName` is displayed  to the user.  |
| `description`        | `string` | optional | Displayed near the prop input. Typically used to contextualize the prop or provide instructions to help users input the correct value. Markdown is supported. |
| `options`        | `[]` or `object[]` or `method` | optional | Provide an array to display options to a user in a drop down menu. Users may select a single option.<br>&nbsp;<br>**`[]` Basic usage**<br>Array of strings. E.g.,<br>`['option 1', 'option 2']`<br>&nbsp;<br>**`object[]` Define Label and Value**<br>`[{ label: 'Label 1', value: 'label1'}, { label: 'Label 2', value: 'label2'}]`<br>&nbsp;<br>**`method` Dynamic Options**<br>You can generate options dynamically (e.g., based on real-time API requests with pagination). See configuration details below. |
| `optional`        | `boolean` | optional | Set to `true` to make this prop  optional. Defaults to `false`. |
| `propDefinition`   | `[]` | optional | Re-use a prop defined in an app file. When you include a prop definition, the prop will inherit values for all the properties listed here. However, you can override those values by redefining them for a given prop instance. See **propDefinitions** below for usage. |
| `default`        | `string` | optional | Define a default value if the field is not completed. Can only be defined for optional fields (required fields require explicit user input) |


#### Referencing User Input Prop Values

| Code        | Description    | Read Scope | Write Scope |
|-------------|----------------|-------------|--------|
| `this.myPropName` | Returns the configured value of the prop | `run()` `hooks` `methods` | n/a (input props may only be modified on component deploy or update via UI, CLI or API) |

#### Advanced Configuration

##### Asyc Options ([example](https://github.com/PipedreamHQ/pipedream/blob/master/components/github/github.app.js))

Async options allow users to select prop values that are programatically generated (e.g., based on an real-time API response).

```javascript
async options({ 
  page,
  prevContext, 
}) {},
```

| Property        | Type    | Required? | Description |
|-------------|----------------|---------------|--------|
| `options()`        | `method` | optional | Typically returns an array of values matching the prop type (e.g., `string`) or an array of object that define the `label` and `value` for each option. The `page` and `prevContext` input parameter names are reserved for pagination (see below).<br>&nbsp;<br>When using `prevContext` for pagination, it must return an object with an `options` array and a `context` object with a `nextPageToken` key. E.g.,  `{ options, context: { nextPageToken }, }` |
| `page` | `integer` | optional |  Returns a `0` indexed page number. For use with APIs that accept a numeric page number for pagination. |
| `prevContext` | `string` | optional | Return a string representing the context for the previous `options` invocation. For use with APIs that accept a token representing the last record for pagination. |



##### Prop Definitions ([example](https://github.com/PipedreamHQ/pipedream/blob/master/components/github/new-commit.js))

Prop definitions enable you to reuse props that are defined in another object. A common use case is to enable re-use of props that are defined for a specific app.

```javascript
props: {
  myPropName: { 
    propDefinition: [
      app, 
      "propDefinitionName", 
      inputValues
    ] 
  },
},

```

| Property        | Type    | Required? | Description |
|-------------|----------------|---------------|--------|
| `propDefinition`        | `array` | optional | An array of options that define a reference to a `propDefinitions` within the `propDefinitions` for an `app` |
| `app` | `object` | required |  An app object |
| `propDefinitionName` | `string` | required | The name of a specific `propDefinition` defined in the corresponding `app` object |
| `inputValues` | `object` | optional | Values to pass into the prop definition. To reference values from previous props, use an arrow function. E.g.,:<br>&nbsp;<br>`c => ({ variableName: c.previousPropName })`|

## Interface Props

Interface props are infrastructure abstractions provided by the Pipedream platform. They declare how a component is invoked — via HTTP request, run on a schedule, etc. — and therefore define the shape of the events it processes.

| Interface Type         | Description                                                                                  | 
|---------------------|----------------------------------------------------------------------------------------------|
| _Timer_ | Invoke your component on an interval (defaults to every hour) or based on a cron expression  | 
| _HTTP_  | Invoke your code on HTTP requests                                                            |

### Timer

To use the timer interface, declare a prop whose value is the string $.interface.timer:

```javascript
props: {
  myPropName: {
    type: "$.interface.timer",
    default: {},
  },
}
```

| Property         | Type | Required? |  Description                                                                                  | 
|---------------------|-------|-----------------------------------------------------------------------------------------|-----|
| `type` | `string` | required | Must be set to `$.interface.timer`  | 
| `default`  | `object` | optional | **Define a default interval**<br>`{ intervalSeconds: 60, },`<br>&nbsp;<br>**Define a default cron expression**<br>` { cron: "0 0 * * *", },`  |

#### Referencing Timer Interface Values

| Code        | Description    | Read Scope | Write Scope |
|-------------|----------------|-------------|--------|
| `this.myPropName` | Returns the type of interface configured (e.g., `{ type: '$.interface.timer' }`) | `run()` `hooks` `methods` | n/a (interface props may only be modified on component deploy or update via UI, CLI or API) |
| `event` | Returns an object with the invocation timestamp and interface configuration (e.g., `{ "timestamp": 1593937896, "interval_seconds": 3600 }`) | `run(event)` | n/a (interface props may only be modified on component deploy or update via UI, CLI or API) |

#### Code Examples

```javascript
props: {
  timer: {
    type: "$.interface.timer"
    default: {
      cron: "0 0 * * *" // Run job once a day
    }
  }
},
```

```javascript
props: {
  timer: {
    type: "$.interface.timer"
    default: {
      intervalSeconds: 60 // Run job once a minute
    }
  }
},
```

### HTTP

To use the HTTP interface, declare a prop whose value is the string $.interface.http:

```javascript
props: {
  myPropName: "$.interface.http",
}
```

| Property         | Type | Required? |  Description                                                                                  | 
|---------------------|-------|-----------------------------------------------------------------------------------------|-----|
| `type` | `string` | required | Must be set to `$.interface.http`  | 
| `respond`  | `method` | required | The HTTP interface exposes a `respond()` method that lets your component issue HTTP responses to the client.  |


#### Referencing HTTP Interface Values

| Code        | Description    | Read Scope | Write Scope |
|-------------|----------------|-------------|--------|
| `this.myPropName` | Returns an object with the unique endpoint URL generated by Pipedream (e.g., `{ endpoint: 'https://abcde.m.pipedream.net' }`) | `run()` `hooks` `methods` | n/a (interface props may only be modified on component deploy or update via UI, CLI or API) |
| `event` | Returns an object representing the HTTP request (e.g., `{ method: 'POST', path: '/', query: {}, headers: {}, bodyRaw: '', body: }`) | `run(event)` | The shape of `event` corresponds with the the HTTP request you make to the endpoint generated by Pipedream for this interface |
| `this.myPropName.respond()` | Returns an HTTP response to the client (e.g., `this.http.respond({status: 200})`) | n/a | `run()` |

#### Responding to HTTP requests
 
The HTTP interface exposes a `respond()` method that lets your component issue HTTP responses. Assuming the prop for your HTTP interface is named `http`:

```javascript
props: {
  http: "$.interface.http"
},
```

you can run `this.http.respond()` to respond to the client from the `run()` method of a component:

```javascript
this.http.respond({
  status: 200,
  headers: {
    "X-My-Custom-Header": "test"
  },
  body: "hello world!" // This can be any string, object, or Buffer
});
```

#### HTTP Event Shape

```javascript
{ 
  method: 'POST', 
  path: '/', 
  query: {}, 
  headers: {}, 
  bodyRaw: '', 
  body: 
}
```

## Service Props

| Service         | Description                                                                                  | 
|---------------------|----------------------------------------------------------------------------------------------|
| _DB_ | Provides access to a simple, component-specific key-value store to maintain state across invocations.| 



### DB

```javascript
props: {
  myPropName: "$.service.db",
}
```

#### Referencing DB Values

| Code        | Description    | Read Scope | Write Scope |
|-------------|----------------|-------------|--------|
| `this.myPropName.get('key')` | Method to get a previously set value for a key. Returns `undefined` if a key does not exist. | `run()` `hooks` `methods` | Use the `set()` method to write values |
| `this.myPropName.set('key', value)` | Method to set a value for a key. Values must be JSON serializable data. | Use the `get()` method to read values | `run()` `hooks` `methods` |


## App Props

```javascript
props: {
  myPropName: {
    type: "app",
    app: "",
    propDefinitions: {}
    methods: {}, 
  },
},
```

| Property        | Type    | Required? | Description |
|-------------|----------------|---------------|--------|
| `type`        | `string` | required | Value must be `app` |
| `app`        | `string` | required | Value must be set to the name slug for an app registered on Pipedream. If you don't see an app listed, please reach out in our public Slack. This data will be discoverable in a self-service way in the near future. |
| `propDefinitions`  | `object` | optional | An object that contains objects with predefined user input props. See the section on User Input Props above to learn about the shapes that can be defined and how to reference in components using the `propDefiniton` property |
| `methods`        | `object` | optional | Define app-specific methods. Methods can be referenced within the app object context via `this` (e.g., `this.methodName()`) and within a component via `this.myAppPropName` (e.g., `this.myAppPropName.methodName()`). |


#### Referencing App Prop Values

| Code        | Description    | Read Scope | Write Scope |
|-------------|----------------|-------------|--------|
| `this.$auth` | Provides access to OAuth tokens and API keys for Pipedream managed auth | **App Object:** `methods` | n/a |
| `this.myAppPropName.$auth` | Provides access to OAuth tokens and API keys for Pipedream managed auth | **Parent Component:** `run()` `hooks` `methods` | n/a |
| `this.methodName()` | Execute a common method defined for an app within the app definition (e.g., from another method) | **App Object:** `methods` | n/a |
| `this.myAppPropName.methodName()` | Execute a common method defined for an app from a component that includes the app as a prop | **Parent Component:** `run()` `hooks` `methods` | n/a |

>**Note:** The specific `$auth` keys supported for each app will be pulished in the near future.

# Methods

You can define helper functions within the `methods` property of your component. You have access to these functions within the [`run` method](#run), or within other methods.

Methods can be accessed using `this.<method-name>`. For example, a `random` method:

```javascript
methods: {
  random() {
    return Math.random()
  },
}
```

can be run like so:

```javascript
const randomNum = this.random();
```

# Hooks

```javascript
hooks: {
  async activate() {},
  async deactivate() {},
},
```

| Property        | Type    | Required? | Description |
|-------------|----------------|---------------|--------|
| `activate`        | `method` | optional | Executed each time a commponent is deployed or updated |
| `deactivate`        | `method` | optional | Executed each time a commponent is deactivated  |

# Dedupe Strategies

> **IMPORTANT:** To use a dedupe strategy, you must emit an `id` as part of the event metadata (dedupe strategies are applied to the submitted `id`)

| Strategy       | Description |
|-------------|----------------|
| `unique`        | Pipedream maintains a cache of 100 emitted `id` values. Events with `id` values that are not in the cache are emitted, and the `id` value is added to the cache. After 100 events, `id` values are purged from the cache based on the order received (first in, first out). A common use case for this strategy is an RSS feed which typically does not exceed 100 items | 
| `greatest`        | Pipedream caches the largest `id` value (must be numeric). Only events with larger `id` values are emitted (and the cache is updated to match the new, largest value). | 
| `last`        | Pipedream caches the ID assocaited with the last emitted event. When new events are emitted, only events after the matching `id` value will be emitted as events. If no `id` values match, then all events will be emitted. | 


# Emitting Events

> `this.$emit()` is a method in scope for the `run` method of a component

```javascript
this.$emit(
  event,
  {
    id,
    summary,
    ts,
  }
)
```

| Property         | Type | Required? |  Description                                                                                  | 
|---------------------|-------|-----------------------------------------------------------------------------------------|-----|
| `event`  | JSON serializable data | optional | The data to emit as the event |
| `id`  | `string` or `number` | Required if a dedupe strategy is applied | A value to uniquely identify this event. Common `id` values may be a 3rd party ID, a timestamp, or a data hash |
| `summary`  | `string` | optional | Define a summary to customize the data displayed in the events list to help differentiate events at a glance  |
| `ts`  | `integer` | optional | If you submit a timestamp, events will automatically be ordered and emitted from oldest to newest. If using the `last` dedupe strategy, the value cached as the `last` event for an invocation will correspond to the event with the newest timestamp. |

# Run

Each time a component is [invoked](#how-components-run) (for example, via HTTP request), its `run` method is called.

The event that triggered the component is passed to `run`, so that you can access it within the method:

```javascript
async run(event) {
  console.log(event)
}
```

You can reference [`this`](#referencing-this) within the `run` method. `this` refers to the component, and provides access to [props](#props), [methods](#methods), and Pipedream-provided functions like `this.$emit`.

You can view logs produced by the `run` method in the **LOGS** section of the Pipedream UI for the component, or using the `pd logs` CLI command:

```bash
pd logs <deployed-component-name>
```

If the `run` method emits events using `this.$emit`, you can access the events in the **EVENTS** section of the Pipedream UI for the component, or using the `pd events` CLI command:

```bash
pd events <deployed-component-name>
```

# Using npm packages

To use an npm package in a component, just require it. There is no `package.json` or `npm install` required.

```javascript
const myVariable = require('npmPackageName')
```

When you deploy a component, Pipedream downloads these packages and bundles them with your deployment.

Some packages — for example, packages like [Puppeteer](https://pptr.dev/), which includes large dependencies like Chromium — may not work on Pipedream. Please [reach out](https://docs.pipedream.com/support/) if you encounter a specific issue.
