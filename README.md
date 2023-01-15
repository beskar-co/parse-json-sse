# parse-json-sse

`parse-json-sse` is a tiny utility for parsing and typing JSON Server-Sent Events (SSE) data. See an example of why we need it [here](https://www.beskar.co/blog/streaming-openai-completions-vercel-edge).

## How does it work?

It uses the `TextDecoder` API to decode the response stream into a string. This is important because the response stream is a `ReadableStream` and we need to convert it to a string to display it in the UI.

The part that's not so straightforward is the response payload. Because we receive Server Sent Events, they arrive in this format (example from OpenAI):

```ts
data: { id: 'test', object: 'test', created: 123, choices: [/* ...*/], model: 'test' }
```

Like, as a string with the `data: ` prefix and everything. Sometimes, we even receive multiple events at once, so we need to split them up and parse them individually. The payload in that state look like this:

```ts
data: { id: 'test', object: 'test', created: 123, choices: [/* ...*/], model: 'test' }

data: { id: 'test', object: 'test', created: 123, choices: [/* ...*/], model: 'test' }
```

This works almost perfectly, but I hit one little snag I didn't expect at all. The response stream behaves as expected when run locally, but when deployed to Vercel, the response stream can return fragmented. Basically we get a response like this:

```ts
data: { id: 'test', object: 'test', created: 123, choices: [{ text: "He
```

Then a moment later, we get the rest of the fragment:

```ts
llo"}], model: 'test' }
```

While I originally thought this might be a bug, Malte Ubl (CTO of Vercel) [noted that](https://twitter.com/cramforce/status/1614304691164438529?s=20&t=8s9FC9XAYQ8A9Ydc85Sjhw):

> This is definitely expected behavior on a busy production system. The fact that local dev happens to flush the buffer whenever the origin flushed the buffer is essentially luck. There might be an npm module that can transform a generic stream into a line based reader

Regardless, this is a problem because we can't parse the JSON until we have the full response. My solution was simply to wrap the `JSON.parse` in a try/catch statement which, if it fails, will shove the response into a temporary state and wait for the next response to come in. When the next response arrives, it prepends the temporary state to the response and tries to parse it again. If it fails again, it just keeps doing that until it succeeds.

## Installation

```bash
yarn add @beskar-labs/parse-json-sse
```

## Usage

```ts
import parseJsonSse from '@beskar-labs/parse-json-sse';

const [text, setText] = useState('');

/* ... */

const response = await fetch('/api/generate', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    prompt,
  }),
});

if (!response.ok) {
  throw new Error(response.statusText);
}

const data = response.body;

if (!data) {
  return;
}

await parseJsonSse<{
  id: string;
  object: string;
  created: number;
  choices?: {
    text: string;
    index: number;
    logprobs: null;
    finish_reason: null | string;
  }[];
  model: string;
}>({
  data,
  onParse: (json) => {
    if (!json.choices?.length) {
      throw new Error('Something went wrong.');
    }

    setText((prev) => prev + json.choices[0].text);
  },
  onFinish: () => {
    console.log('Finished!');
  },
});
```
