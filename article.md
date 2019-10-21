---
title: Mocking API calls in React Native tests
published: false
description: 'TLDR: Use unmock and react-native-testing-library'
tags: javascript, reactnative, testing
---

Testing networking logic in React Native apps can be hard. You don't want to use the production API to run tests, so you need to mock network calls. Mocking also allows you to test not only the happy-land case where everything works smoothly, but also the case where the API breaks horribly.

There are different ways to mock network calls. One could use dependency injection to inject "fetching service" into the components and replace the service with a mock in tests. Alternatively, one could use [Context](https://reactjs.org/docs/context.html) to wrap components in a fetching service context. Both of these solutions can work, but there should be a simpler way.

In this post, we are going to build a simple React Native application that's tested in end-to-end fashion with mock data served by [unmock](https://www.unmock.io). Unmock is an HTTP test library that uses [node-mitm](https://github.com/moll/node-mitm) behind the scenes to intercept all calls made by your app. At intercept, it either returns fake data adhering to the schema of the API or, in case it encounters an unexpected call, raises an error.

We'll run our tests in Node.js environment using [Jest](https://jestjs.io/), with [react-native-testing-library](https://github.com/callstack/react-native-testing-library) used to render the app component with all the React hook sugar included.

You can find the repository for this project [here](https://github.com/unmock/unmock-react-native-example). Instructions for running the app are contained in the repository.

## Tour of the app

The [example application](https://github.com/unmock/unmock-react-native-example) shows a random cat fact fetched from the [Cat Facts API](https://alexwohlbruck.github.io/cat-facts/). User can refresh the fact by pressing the button. The app in all its glory looks like this, running here in Android virtual device:

<p align="center">
<img src="./screenshot.png" alt="drawing" width="200"/>
</p>

Code for the app contains a single component defined in [App.tsx](https://github.com/unmock/unmock-react-native-example/blob/master/src/App.tsx). At high-level, we define the `App` component like this:

```ts
const App = () => {
  /* React hooks */
  const [shownFact, setFact] = useState('');
  const [err, setError] = useState(null);
  const [loading, setLoading] = useState(false);

  /* Refresh cat fact, see below */
  const refreshFact = async () => {
    /* */
  };

  /* Initial data fetching */
  useEffect(() => {
    refreshFact();
  }, []);

  return (
    /* JSX, see below */
  );
};
```

We use `useState` from React hooks for state management. The variables `shownFact`, `err`, and `loading` are used to contain the shown cat fact, possible fetch error, and the loading state.

Refreshing the shown fact at start-up or when user presses the button is made via `refreshFact` function defined in `App`:

```ts
const refreshFact = async () => {
  try {
    setLoading(true);
    const fact = await fetchFact();
    setFact(fact);
    setError(null);
  } catch (err) {
    setError(err);
  } finally {
    setLoading(false);
  }
};
```

This function sets the component state as appropriate and relies on `fetchFact` function to do the network call. This function uses the [Fetch API](https://facebook.github.io/react-native/docs/network) provided by React Native:

```ts
const CAT_FACT_URL =
  'https://cat-fact.herokuapp.com/facts/random?animal_type=cat&amount=1';

const fetchFact = async () => {
  const fetchResult = await fetch(CAT_FACT_URL);
  if (!fetchResult.ok) {
    throw Error(`Failed fetching cat fact with code: ${fetchResult.status}`);
  }
  const body = await fetchResult.json();
  const fact = body.text;
  return fact;
};
```

We parse the cat fact by parsing a JSON from the response body and accessing the cat fact in the `text` field, as documented [here](https://alexwohlbruck.github.io/cat-facts/docs/endpoints/facts.html).

Content is rendered based on the values of `loading` and `err`:

```jsx
{
  loading ? (
    <Text style={styles.loading} testID="loading">
      Loading...
    </Text>
  ) : err ? (
    <Text style={{...styles.fact, ...styles.error}} testID="error">
      Something went horribly wrong, please try again!
    </Text>
  ) : (
    <Text style={styles.fact} testID="fact">
      {shownFact}
    </Text>
  );
}
```

If the state of `loading` is `true`, we show the text "Loading...". If the state of `err` contains an error, apology is shown to the user. Otherwise, the cat fact is rendered.

Note that we also give the components [testID](https://facebook.github.io/react-native/docs/view#testid) properties to simplify testing.

## Writing tests

### Prerequisites

Tests for the application are found in [App.test.tsx](https://github.com/unmock/unmock-react-native-example/blob/master/__tests__/App.test.tsx).

The first step in the tests is to fill in `fetch` (not natively available in Node.js) with [node-fetch](https://www.npmjs.com/package/node-fetch):

```ts
// @ts-ignore
global.fetch = require('node-fetch');
```

In the `beforeAll` block, we switch on Unmock to intercept all outgoing traffic and define the API behaviour with the [nock](https://www.unmock.io/docs/unmock)-like syntax:

```ts
beforeAll(() => {
  unmock.on();
  unmock
    .nock('https://cat-fact.herokuapp.com', 'catFactApi')
    .get('/facts/random?animal_type=cat&amount=1')
    .reply(200, {text: u.string('lorem.sentence')})
    .reply(500, 'Internal server error');
});
```

In `unmock.nock` call, we set the URL for the mocked service as well as give the name `catFactApi` for the created fake service. Below, we use the `catFactApi` to control the "state" of the service

After specifying the mock API URL, we then specify the mocked operation: for a `GET` request to the given URL, the API should either return a `200` response with a JSON body or throw an internal server error. The syntax `{ text: u.string('lorem.sentence') }` means that the response body should be a JSON object with `text` property. The value for the property should be a random string generated with `lorem.sentence` API from [faker.js](https://github.com/marak/Faker.js/). Notice how we don't need to hardcode "foo" or "bar" in our tests!

Finally, we reset the state of `unmock` before each test so that the tests remain independent:

```ts
beforeEach(() => {
  unmock.reset();
});
```

### Test for success

The first test ensures that when the API successfully returns a cat fact, the fact block is displayed to the user:

```ts
it('renders the fact block when API succeeds', async () => {
  const api = unmock.services['catFactApi'];
  api.state(transform.withCodes(200));
  const renderApi: RenderAPI = render(<App />);

  await waitForElement(() => {
    return renderApi.getByTestId('fact');
  });
});
```

First we set the API to always return 200, simulating success. We then use `render` from `react-native-testing-library` to fully render the component and run all hooks. We then use `waitForElement` from the same library to wait for the element with `testID="fact"` to show up.

Second test for success ensures that when user clicks the button, new fact is fetched from API. The button press is simulated with the `fireEvent` from `react-native-testing-library`:

```ts
it('renders new fact after clicking the button', async () => {
  const api = unmock.services['catFactApi'];
  api.state(transform.withCodes(200));

  const renderApi: RenderAPI = render(<App />);

  fireEvent.press(renderApi.getByText('Get me a new one'));

  await waitForElement(() => {
    const secondCall = api.spy.secondCall;
    const secondFact = secondCall.returnValue.bodyAsJson;
    return renderApi.getByText(secondFact.text);
  });
});
```

Here we again use `waitForElement`, but instead of relying on an element with `testID="fact"` to show up, we wait for an element that contains the same text as the (random) fact returned from the API.

Because we're returning a random string from the API (via `lorem.sentence`), we need here to access the random value returned from the service. Unmock services keep track of their calls and return values in the `spy` property, which is a [SinonJS spy](https://sinonjs.org/releases/latest/spies/). The second spy call is accessed via the `secondCall` property and its return value via `returnValue`.

### Test for failure

Test for failure proceeds similarly as the test for success: we change the API to return status code 500, render the app, and wait for the element with `testID="error"` to show up.

```ts
it('renders error when the API fails', async () => {
  const api = unmock.services['catFactApi'];
  api.state(transform.withCodes(500));

  const renderApi: RenderAPI = render(<App />);

  await waitForElement(() => {
    return renderApi.getByTestId('error');
  });
});
```

### Conclusion

That's it! Using Unmock, Jest and `react-native-testing-library`, we wrote comprehensive integration tests for our application component. The tests running in Node.js made sure that React hooks are properly triggered and that the returned data is rendered as expected, both in the case when the API call succeeds and when it fails. We did not need to inject extra dependencies into our component nor hack with React context.

Note that `unmock` currently only supports Node.js environment. If you would like to see Unmock run in React Native and populate your app with fake data, maybe create an issue in [unmock-js](https://github.com/unmock/unmock-js) repository 😊

Thanks a lot for reading, as always we appreciate any feedback and comments!