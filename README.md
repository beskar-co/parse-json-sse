# parse-json-sse

`parse-json-sse` is a tiny utility for parsing and typing JSON Server-Sent Events (SSE) data. See an example of why we need it [here](https://www.beskar.co/blog/streaming-openai-completions-vercel-edge).

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
