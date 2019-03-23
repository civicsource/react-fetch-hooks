# React Fetch Hooks

> A set of react hooks to work with the fetch API and gracefully parse & deal with HTTP errors.

```js
const { isFetching, isFetched, error, data } =
		useFetch(`https://api.example.com/`);

const { isFetching, isFetched, error, data: result, fetch: saveThing } =
		useLazyFetch({
			url: `https://api.example.com/`,
			method: "POST"
		});

// ...later, maybe in response to a user action
saveThing();
```

See more [in-depth examples](#examples).

## Install

Install with [Yarn](https://yarnpkg.com/en/):

```
yarn add react-fetch-hooks
```

or if Yarn isn't your thing:

```
npm install react-fetch-hooks --save
```

### Polyfill the Browser

[See here for a browser polyfill](https://github.com/github/fetch) if you are using the [fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) in a browser that doesn't support it yet.

### Polyfill the Server

If using this library in node, make use of the [`node-fetch` library](https://github.com/bitinn/node-fetch) to polyfill `fetch`:

```js
global.fetch = require("node-fetch");
```

Do that at the beginning of the entry point for your app and then you can use `react-fetch-hooks` as normal.

## Examples

```js
import React from "react";
import { useFetch } from "react-fetch-hooks";

const MyBanana = ({ id }) => {
	const { isFetching, isFetched, error, data: banana } =
		useFetch(`https://api.example.com/bananas/${id}`);

	if (isFetching) {
		return <span>Loading...</span>;
	}

	if (error) {
		// the error message will be parsed from the HTTP response, if available
		return <span>Some shit broke: {error}</span>;
	}

	return <span>My banana is {banana.color}!</span>;
};
```

This will make a `GET` request to `api.example.com` whenever the `id` prop passed to the component changes. It will then return the status of the data being loaded as well as the data when it is ready. The `useFetch` hook takes all the same parameters/options as the [standard fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API).

### Lazy Evaluation

You can also use the hook to lazily create a function that can be invoked later:

```js
import React from "react";
import { useLazyFetch } from "react-fetch-hooks";

const BananaEditor = ({ color }) => {
	const { isFetching: isSaving, error, fetch: saveBanana } =
		useLazyFetch({
			url: "https://api.example.com/bananas/",
			method: "POST",
			body: JSON.stringify({ color })
		});

	if (isSaving) {
		return <span>Saving...</span>;
	}

	if (error) {
		return <span>Some shit broke: {error}</span>;
	}

	return <button onClick={saveBanana}>Save new banana with color: {color}</button>;
};
```

This will not invoke the fetch until the button is clicked.

### Polling on an Interval

You can pass a `refreshInterval` parameter to any of the fetch hooks to refresh the data (do the fetch again) after a certain amount of time has passed. This could be useful to poll the server changes on a set interval:

```js
import React from "react";
import { useFetch } from "react-fetch-hooks";

const MyBanana = ({ id }) => {
	const { data: banana } = useFetch({
		url: `https://api.example.com/bananas/${id}`,
		refreshInterval: 10000
	});

	if(!banana) return null;
	return <span>My banana is {banana.color}!</span>;
};
```

This will make a `GET` request and update the data from the server every 10 seconds.

### Reset Fetched Data

You can pass a `resetDelay` parameter to any of the fetch hooks to reset the data after a certain amount of time has passed. This could be useful to clear the "saved" flag after a short amount of time:

```js
import React from "react";
import { useLazyFetch } from "react-fetch-hooks";

const BananaEditor = ({ color }) => {
	const { isFetching: isSaving, isFetched: isSaved, fetch: saveBanana } =
		useLazyFetch({
			url: "https://api.example.com/bananas/",
			method: "POST",
			body: JSON.stringify({ color }),
			resetDelay: 3000
		});

	if (isSaving) {
		return <span>Saving...</span>;
	}

	if (isSaved) {
		return <span>You just saved a great banana! Congrations to you!</span>;
	}

	return <button onClick={saveBanana}>Save new banana with color: {color}</button>;
};
```

This will clear the `isSaved` flags 3 seconds after the `POST` finishes successfully.

### Conditional Fetching

Since you [shouldn't wrap your hooks in conditionals](https://reactjs.org/docs/hooks-rules.html), if you want to conditionally fetch, you can pass a `null` or empty URL to the hook which will tell it not to do anything:

```js
import React from "react";
import { useFetch } from "react-fetch-hooks";

const MyBanana = ({ id }) => {
	const { data: banana } = useFetch(id ? `https://api.example.com/bananas/${id}` : null);

	if (!banana) return null;
	return <span>My banana is {banana.color}!</span>;
};
```

This will only make the `GET` request when a valid `id` has been passed to the component. Any of the following will also accomplish the same result:

```js
// all of these will do nothing
useFetch();
useFetch("");
useFetch({
	url: null,
	headers: {
		"Content-Type": "application/json"
	}
});
useFetch({
	// missing URL
	headers: {
		"Content-Type": "application/json"
	}
});
```

### Bearer Tokens

You can pass specific headers to the hook but setting an _Authorization_ header with a `bearerToken` is one we use often enough that we built in a shortcut for it:

```js
import React from "react";
import { useFetch } from "react-fetch-hooks";

const MyBanana = ({ id, authToken = "mytoken" }) => {
	const { data: banana } = useFetch({
		url: `https://api.example.com/bananas/${id}`,
		bearerToken: authToken
	});

	if (!banana) return null;
	return <span>My banana is {banana.color}!</span>;
};
```

This will make a `GET` request adding an `Authorization` header with the value `Bearer mytoken`.

## Utility Functions

The package includes some other utility functions that can be used outside the context of a hook.

### `checkStatus(response)`

[Read here](https://github.com/github/fetch#handling-http-error-statuses) for the inspiration for this function. It will reject fetch requests on any non-2xx response. It differs from the example in that it will try to parse a JSON body from the non-200 response and will set any `message` field (if it exists) from the JSON body as the error message. The fetch hooks use this internally.

```js
import { checkStatus } from "react-fetch-hooks";

//given a 400 Bad Request response with a JSON body of:
//{ "message": "Invalid arguments. Try again.", "someOtherThing": 42 }

fetch("/data", {
	method: "GET",
	headers: {
		Accept: "application/json"
	}
})
.then(checkStatus)
.catch(err => {
	console.log(err.message); //Invalid Arguments. Try again.
	console.log(err.response.statusText); //Bad Request
	console.log(err.response.jsonBody); //{ "message": "Invalid arguments. Try again.", "someOtherThing": 42 }
});
```

It will try to look for a `message` field first, and then an `exceptionMessage` falling back to the `statusText` if neither one exist or if the response body is not JSON.

## Build/Run Locally

After cloning this repo, run:

```
yarn
yarn lint
yarn test
yarn compile
```
