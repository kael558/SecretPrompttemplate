# Building a Feedback Generator with LLMs

This project demonstrates how to build a structured **feedback generator** using Large Language Models (LLMs). The design is modular:

1. **Chat function** – handles all interactions with the language model API.
2. **Generator function** – defines the system prompt, prepares data, and asks the model for a response.
3. **Parser function** – interprets the raw LLM output into structured feedback and scores.

This separation of concerns makes the system flexible, debuggable, and easy to extend.

---

## Abstract Design

### 1. Chat Function (LLM Gateway)

The chat function is a **generic wrapper** around API calls to different providers (OpenAI, Groq, Anthropic, etc.).
It:

* Accepts messages in the `{ role, content }` format.
* Passes them to the model.
* Returns the raw string response from the model.
* Handles **fallbacks** (e.g., retrying with OpenAI if Groq is rate limited).

**Abstract Pseudocode:**

```js
async function chat(messages, model, options) {
  // Clean and prepare messages
  // Add special response format instructions if needed (e.g., JSON mode)

  try {
    // Send request to primary provider (Groq, OpenAI, etc.)
    // If fails, retry with fallback
    // Return the raw LLM response (string)
  } catch (error) {
    // Handle network errors, timeouts, retries
  }
}
```

---

### 2. Generator Function (Task Logic)

The generator function defines **what you want the LLM to do**.
It:

* Defines the **system prompt** (instructions).
* Prepares a **conversation transcript**.
* Sends everything to the chat function.
* Returns parsed results.

**Abstract Pseudocode:**

```js
async function generateFeedbackContent(messages, scenario) {
  // Build the system prompt with task description
  // Transform conversation into numbered format
  // Wrap everything into messages for the LLM

  // Send to chat function
  const rawResponse = await chat(formattedMessages, chosenModel);

  // Parse the LLM output into structured feedback
  return parseFeedback(rawResponse);
}
```

---

### 3. Parser Function (Output Interpreter)

The parser ensures the model’s response is **usable by your application**.
It:

* Extracts feedback and scores.
* Validates formatting.
* Optionally retries if the response isn’t valid.

**Abstract Pseudocode:**

```js
function parseFeedback(response) {
  // Identify conversation feedback
  // Extract numbered feedback
  // Extract scoring categories with justifications
  // Return structured JSON or object
}
```

---

## Concrete Implementation

Here’s how this looks in code.

### `chat.js` – The Chat Function

```js
export async function chat(
  messages,
  model = "gemma2-9b-it",
  json_mode = false
) {
  const cleanedMessages = messages.map(({ role, content }) => ({ role, content }));

  const payload = { model, messages: cleanedMessages };
  if (json_mode) payload.response_format = { type: "json_object" };

  try {
    // Special case for OpenAI model
    if (payload.model === "gpt-4o-mini") {
      const openaiResponse = await fetch("https://api.openai.com/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
        },
        body: JSON.stringify(payload),
      });
      const data = await openaiResponse.json();
      return data.choices[0].message.content.replace(/\*/g, "");
    }

    // Default: Try Groq first
    const groqResponse = await fetch("https://api.groq.com/openai/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${process.env.GROQ_API_KEY}`,
      },
      body: JSON.stringify(payload),
    });

    if (!groqResponse.ok) {
      console.log("Rate limited by Groq, trying OpenAI...");
      payload.model = "gpt-4o-mini";

      const openaiResponse = await fetch("https://api.openai.com/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
        },
        body: JSON.stringify(payload),
      });
      const data = await openaiResponse.json();
      return data.choices[0].message.content.replace(/\*/g, "");
    }

    const data = await groqResponse.json();
    return data.choices[0].message.content.replace(/\*/g, "");
  } catch (error) {
    console.error("Error:", error);
    throw error;
  }
}
```

---

### `generator.js` – The Feedback Generator

This is where the **system prompt** lives.

```js
export async function generateFeedbackContent(messages, scenario) {
  const { task, description, additional_context, inputMode, outputMode, inputScore, outputScore } = scenario;

  const system_msg = `# INSTRUCTIONS

You are an English language tutor...
(Task: ${task}, Description: ${description}, etc.)
`;

  messages.shift(); // remove system message if present

  let user_response_index = 1;
  const conversation = messages.map((message) => {
    if (message.role === "user") {
      return `User response #${user_response_index++}: ${message.content}`;
    }
    return `${message.role}: ${message.content}`;
  }).join("\n");

  const formattedMessages = [
    { role: "system", content: system_msg },
    { role: "user", content: `## Conversation\n${conversation}\n\nOutput the feedback and scores.` }
  ];

  let retries = 0;
  const maxRetries = 3;

  while (retries < maxRetries) {
    try {
      const response = await chat(formattedMessages, "llama-3.1-70b-versatile");
      const result = parseFeedback(response);
      return result;
    } catch (error) {
      retries++;
      console.error(error);

      if (retries < maxRetries) {
        formattedMessages.push({ role: "user", content: error.message });
        await new Promise((resolve) => setTimeout(resolve, 5000));
      }
    }
  }

  throw new Error(`Failed to generate feedback content after ${maxRetries} attempts.`);
}
```

---

### `parser.js` – The Parser (Simplified Example)

```js
export function parseFeedback(response) {
  // In practice, you'd use regex or structured output
  // For now, assume LLM followed instructions
  return {
    raw: response,
    feedback: extractFeedback(response),
    scores: extractScores(response),
  };
}

function extractFeedback(response) {
  // regex to find lines starting with numbers
  return response.match(/\d+\.\s[^\n]+/g) || [];
}

function extractScores(response) {
  // regex to find lines like "Grammar: 70/100 - ..."
  return response.match(/[A-Za-z ]+: \d+\/100 - .+/g) || [];
}
```

---

## How to Use

1. Clone the repo.
2. Install dependencies:

   ```bash
   npm install
   ```
3. Add your API keys to `.env`:

   ```env
   OPENAI_API_KEY=your-key
   GROQ_API_KEY=your-key
   ```
4. Run the generator with a sample conversation and scenario:

   ```js
   import { generateFeedbackContent } from "./generator.js";

   const messages = [
     { role: "assistant", content: "Hello! How can I help you today?" },
     { role: "user", content: "I want join cooking class." },
     // ...
   ];

   const scenario = {
     task: "Joining a class",
     description: "User registers for a cooking class",
     additional_context: "",
     inputMode: "Speaking",
     outputMode: "Listening",
     inputScore: 4,
     outputScore: 5,
   };

   const feedback = await generateFeedbackContent(messages, scenario);
   console.log(feedback);
   ```

---

## Key Takeaways

* **Separate responsibilities**:

  * Chat = API interaction
  * Generator = task definition
  * Parser = result cleaning

* **Robustness**: Retry on failures, fallback between providers.

* **Reusability**: You can plug in any scenario (job interviews, customer support, classroom roleplays) without changing the core functions.

---

👉 This pattern is useful beyond language tutoring. You can apply it to **code review bots, interview simulations, customer service QA, or structured content generation**.
