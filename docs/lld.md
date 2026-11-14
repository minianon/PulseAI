# PulseAI: Low-Level Design

For the big picture see the [High-Level Design](hld.md).

## 1. Firestore collections

| Collection | Document fields |
|---|---|
| `users` | name, email |
| `interviews` | role, type, level, techstack, questions[], userId, finalized, coverImage, createdAt |
| `feedback` | interviewId, userId, totalScore, categoryScores[], strengths[], areasForImprovement[], finalAssessment, createdAt |

## 2. Feedback schema (zod `feedbackSchema`)

```mermaid
classDiagram
  class Feedback {
    totalScore: number
    categoryScores: CategoryScore[]
    strengths: string[]
    areasForImprovement: string[]
    finalAssessment: string
  }
  class CategoryScore {
    name: string
    score: number (0-100)
    comment: string
  }
  Feedback --> CategoryScore
```

The LLM is called with `generateObject({ schema: feedbackSchema })`, so the result always matches this shape.

## 3. Server actions

| Action | Behavior |
|---|---|
| `signUp`, `signIn`, `signOut` | Firebase user record; `setSessionCookie` creates a session cookie valid for one week |
| `getCurrentUser`, `isAuthenticated` | Verify the session cookie with Firebase Admin |
| `getInterviewsByUserId`, `getLatestInterviews`, `getInterviewById` | Firestore queries |
| `createFeedback` | Formats the transcript, calls `generateObject`, saves to `feedback` |
| `getFeedbackByInterviewId` | Loads feedback for the feedback page |

## 4. Interview generation

```mermaid
sequenceDiagram
  actor C as Candidate
  participant V as Vapi assistant
  participant W as /api/vapi/webhook
  participant G as /api/vapi/generate
  participant AI as Gemini
  participant DB as Firestore
  C->>V: talk: role, level, tech stack, number of questions
  V->>W: tool call generate_questions
  W->>G: POST { type, role, level, techstack, amount, userid }
  G->>G: 400 if role, amount or userid missing
  G->>AI: generateText(questions prompt, JSON array only)
  AI-->>G: ["Question 1", ...]
  G->>DB: add interviews document
  G-->>W: { success, interviewId }
  W-->>V: tool result
```

## 5. Interview and feedback

```mermaid
sequenceDiagram
  actor C as Candidate
  participant A as Agent component
  participant V as Vapi
  participant S as createFeedback
  participant AI as Gemini
  C->>A: Call
  A->>V: vapi.start(assistantId, { username, userid, questions })
  loop conversation
    V-->>A: message events (final transcripts)
  end
  C->>A: Finish (or call ends)
  A->>S: createFeedback({ interviewId, userId, transcript })
  S->>AI: generateObject(feedbackSchema)
  S-->>A: { success, feedbackId }
  A->>A: router.push(/interview/:id/feedback)
```

Feedback is generated once per call (guarded by a ref), whether the call ends on its own or the user clicks Finish.

## 6. Configuration

| Variable | Purpose |
|---|---|
| `NEXT_PUBLIC_VAPI_PUBLIC_KEY`, `NEXT_PUBLIC_VAPI_ASSISTANT_ID` | Vapi client and assistant |
| `NEXT_PUBLIC_BASE_URL` | Used by the webhook to call `/api/vapi/generate` |
| `GOOGLE_GENERATIVE_AI_API_KEY` | Gemini |
| `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` | Firebase Admin (the web client config is in `Firebase/client.ts`) |
