# The Three-Part Pattern for Robust LLM Development

When building applications with Large Language Models, I've found that separating concerns into three distinct functions creates more maintainable, reliable, and testable code. This pattern has served me well across dozens of production applications.

## The Abstract Pattern

### 1. The Chat Function (Infrastructure Layer)
**Purpose**: Handle all communication with language models, including error handling, retries, and fallbacks.

**Responsibilities**:
- API communication and authentication
- Error handling and retry logic
- Provider fallbacks (e.g., Groq → OpenAI)
- Rate limit management
- Response cleaning and normalization

**Key Principle**: This function should be provider-agnostic and focus purely on reliable message delivery.

### 2. The Generator Function (Business Logic Layer)
**Purpose**: Define the specific task, context, and requirements for the language model.

**Responsibilities**:
- Construct system prompts with domain knowledge
- Format input data appropriately
- Define output requirements and constraints
- Handle task-specific retry logic
- Orchestrate the conversation flow

**Key Principle**: This function contains all the domain expertise and prompt engineering for your specific use case.

### 3. The Parser Function (Data Processing Layer)
**Purpose**: Transform raw LLM output into structured, validated data your application can use.

**Responsibilities**:
- Parse and validate LLM responses
- Handle malformed or unexpected outputs
- Convert to application-specific data structures
- Provide meaningful error messages for failures

**Key Principle**: This function should be defensive and handle the unpredictable nature of LLM outputs gracefully.

## Benefits of This Pattern

- **Separation of Concerns**: Each function has a single, clear responsibility
- **Testability**: Each layer can be tested independently
- **Reusability**: The chat function can be reused across different use cases
- **Maintainability**: Changes to prompts, parsing, or providers are isolated
- **Reliability**: Centralized error handling and fallback strategies

---

## Concrete Implementation

Let's build a simple email classification system that categorizes customer emails as "support", "sales", or "billing".

### 1. The Chat Function - Infrastructure Layer

```javascript
export async function chat(messages, options = {}) {
    const {
        model = "llama-3.1-70b-versatile",
        maxRetries = 3,
        retryDelay = 1000
    } = options;

    const payload = {
        model,
        messages: messages.map(({ role, content }) => ({ role, content }))
    };

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            // Try primary provider (Groq)
            const groqResponse = await fetch("https://api.groq.com/openai/v1/chat/completions", {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    "Authorization": `Bearer ${process.env.GROQ_API_KEY}`
                },
                body: JSON.stringify(payload)
            });

            if (groqResponse.ok) {
                const data = await groqResponse.json();
                return data.choices[0].message.content;
            }

            // Fallback to OpenAI if Groq fails
            console.log(`Groq failed (attempt ${attempt}), trying OpenAI...`);
            
            const openaiPayload = { ...payload, model: "gpt-4o-mini" };
            const openaiResponse = await fetch("https://api.openai.com/v1/chat/completions", {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                    "Authorization": `Bearer ${process.env.OPENAI_API_KEY}`
                },
                body: JSON.stringify(openaiPayload)
            });

            if (openaiResponse.ok) {
                const data = await openaiResponse.json();
                return data.choices[0].message.content;
            }

            throw new Error(`Both providers failed on attempt ${attempt}`);

        } catch (error) {
            console.error(`Attempt ${attempt} failed:`, error.message);
            
            if (attempt === maxRetries) {
                throw new Error(`All ${maxRetries} attempts failed: ${error.message}`);
            }

            // Wait before retrying
            await new Promise(resolve => setTimeout(resolve, retryDelay * attempt));
        }
    }
}
```

### 2. The Generator Function - Business Logic Layer

```javascript
export async function classifyEmail(emailContent, senderEmail = null) {
    const systemPrompt = `You are an expert email classifier for a customer service team.

Your task is to classify incoming customer emails into exactly one of these categories:
- "support" - Technical issues, bug reports, how-to questions
- "sales" - Pricing inquiries, product questions, purchase interest  
- "billing" - Payment issues, invoice questions, subscription changes

Guidelines:
- Always respond with exactly one word: support, sales, or billing
- If unclear, choose the most likely category based on context
- Consider the sender's email domain for additional context

Examples:
- "My app keeps crashing when I try to upload files" → support
- "What's the difference between your Pro and Enterprise plans?" → sales  
- "I was charged twice this month" → billing`;

    const userMessage = senderEmail 
        ? `Email from: ${senderEmail}\n\nContent: ${emailContent}`
        : `Content: ${emailContent}`;

    const messages = [
        { role: "system", content: systemPrompt },
        { role: "user", content: userMessage }
    ];

    try {
        const response = await chat(messages, { 
            model: "llama-3.1-70b-versatile",
            maxRetries: 3 
        });
        
        return parseClassification(response);
    } catch (error) {
        console.error("Failed to classify email:", error);
        throw new Error("Email classification failed");
    }
}
```

### 3. The Parser Function - Data Processing Layer

```javascript
function parseClassification(response) {
    if (!response || typeof response !== 'string') {
        throw new Error("Invalid response: expected string");
    }

    // Clean and normalize the response
    const cleaned = response.trim().toLowerCase();
    
    // Valid categories
    const validCategories = ['support', 'sales', 'billing'];
    
    // Check for exact match
    if (validCategories.includes(cleaned)) {
        return cleaned;
    }
    
    // Check if response contains a valid category
    for (const category of validCategories) {
        if (cleaned.includes(category)) {
            return category;
        }
    }
    
    // Handle common variations
    const variations = {
        'tech': 'support',
        'technical': 'support', 
        'help': 'support',
        'bug': 'support',
        'sell': 'sales',
        'buy': 'sales',
        'purchase': 'sales',
        'price': 'sales',
        'payment': 'billing',
        'invoice': 'billing',
        'charge': 'billing',
        'subscription': 'billing'
    };
    
    for (const [variation, category] of Object.entries(variations)) {
        if (cleaned.includes(variation)) {
            return category;
        }
    }
    
    throw new Error(`Could not parse classification from response: "${response}"`);
}
```

### 4. Putting It All Together

```javascript
// Usage example
async function handleIncomingEmail(emailContent, senderEmail) {
    try {
        const category = await classifyEmail(emailContent, senderEmail);
        
        console.log(`Email classified as: ${category}`);
        
        // Route to appropriate team
        switch (category) {
            case 'support':
                return routeToSupportTeam(emailContent, senderEmail);
            case 'sales':
                return routeToSalesTeam(emailContent, senderEmail);
            case 'billing':
                return routeToBillingTeam(emailContent, senderEmail);
        }
    } catch (error) {
        console.error("Email processing failed:", error);
        // Route to general queue as fallback
        return routeToGeneralQueue(emailContent, senderEmail);
    }
}

// Example usage
const email = "Hi, I'm interested in your enterprise pricing. Can you send me a quote?";
const category = await classifyEmail(email, "john@company.com");
// Returns: "sales"
```

## Key Takeaways

1. **Keep functions focused**: Each function has one clear responsibility
2. **Plan for failure**: LLMs are unpredictable; always have fallbacks and error handling
3. **Make it testable**: You can unit test each function independently
4. **Provider flexibility**: The chat function abstracts away which LLM you're using
5. **Graceful degradation**: If classification fails, the system still functions

This pattern scales beautifully from simple classification tasks to complex multi-turn conversations, making your LLM applications more reliable and maintainable.

---

*Follow me on LinkedIn for more practical AI development patterns and insights from building production LLM applications.*
