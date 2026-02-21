# PlacementBot Backend Implementation Guide

This guide shows how to implement the PlacementBot backend API using Express.js and OpenRouter.

## Prerequisites

```bash
npm install express
npm install --save-dev @types/express typescript
```

## Backend Route Implementation

Create a new file `routes/placementBotRoute.ts` in your backend project:

```typescript
// Backend API route handler for PlacementBot using OpenRouter
// This should be placed in your backend/API layer

import express, { Request, Response } from 'express';

const router = express.Router();
const OPENROUTER_API_KEY = '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';

const SYSTEM_PROMPT = `You are PlacementBot, an expert AI Career Assistant for college placement. You help students with:
- Checking eligibility for companies based on their CGPA, skills, and branch
- Mock interview preparation with realistic questions
- Resume improvement tips and ATS optimization
- Skill gap analysis and learning recommendations
- Placement statistics and company insights
- Career guidance and job market trends

Be helpful, encouraging, and provide specific, actionable advice. Keep responses concise and friendly. Use bullet points when listing items.`;

/**
 * POST /api/placement-bot/chat
 * Handle chat messages from PlacementBot
 */
router.post('/api/placement-bot/chat', async (req: Request, res: Response): Promise<void> => {
  try {
    const { message, history } = req.body;

    if (!message) {
      return res.status(400).json({ error: 'Message is required' });
    }

    // Build messages array
    const messages = [
      { role: 'system', content: SYSTEM_PROMPT },
      ...(history || []),
      { role: 'user', content: message }
    ];

    // Call OpenRouter API
    const response = await fetch('https://openrouter.io/api/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${OPENROUTER_API_KEY}`,
        'HTTP-Referer': 'https://placementpro.local',
        'X-Title': 'PlacementPro X'
      },
      body: JSON.stringify({
        model: 'openai/gpt-3.5-turbo',
        messages: messages,
        temperature: 0.7,
        max_tokens: 1000,
        top_p: 0.95
      })
    });

    if (!response.ok) {
      const error = await response.json();
      console.error('OpenRouter API Error:', error);
      return res.status(response.status).json({
        error: error.error?.message || 'Failed to get response from AI'
      });
    }

    const data = await response.json();
    const assistantMessage = data.choices[0]?.message?.content || 'I apologize, but I could not generate a response.';

    res.json({
      success: true,
      response: assistantMessage,
      usage: data.usage
    });
  } catch (error) {
    console.error('PlacementBot API Error:', error);
    res.status(500).json({
      error: 'Internal server error',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

export default router;
```

## Integration in Express App

```typescript
import express from 'express';
import placementBotRoute from './routes/placementBotRoute';

const app = express();

app.use(express.json());
app.use(placementBotRoute);

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`PlacementBot API running on port ${PORT}`);
});
```

## Frontend Integration

The frontend already has the OpenRouter integration setup. When moving to a backend, update the API call in your frontend code:

```typescript
// Old: Direct OpenRouter call from frontend
const response = await getOpenRouterResponse(prompt, history);

// New: Use your backend endpoint
const response = await fetch('/api/placement-bot/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ message: prompt, history })
});
const data = await response.json();
```

## API Request/Response Format

**Request:**
```json
{
  "message": "Am I eligible for Google?",
  "history": [
    { "role": "user", "content": "Hi" },
    { "role": "assistant", "content": "Hello!" }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "response": "Based on your profile...",
  "usage": {
    "prompt_tokens": 150,
    "completion_tokens": 100
  }
}
```

## Environment Variables

```env
OPENROUTER_API_KEY=sk-or-v1-...
BACKEND_URL=http://localhost:3001
```

## Notes

- Never expose the API key in frontend code (use backend instead)
- Set up proper authentication/authorization
- Implement rate limiting for production
- Add request validation and sanitization
- Consider caching responses for common queries
