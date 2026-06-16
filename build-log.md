# AI Conversation Logs — Cal AI Clone (Calorie Tracker)

> **Project:** AI-Powered Calorie Tracker  
> **Builder:** Lohith  
> **Log Format:** Prompt → AI Response (summarized)  
> **Tool Used:** Claude (claude-sonnet-4-6)

---

## Session 1 — Project Setup & Architecture

**Date:** 2026-06-10

---

**[USER]**
I want to build a Cal AI clone — an AI-powered calorie tracker where users can log meals by taking a photo or typing a description. What stack should I use?

**[AI]**
For a Cal AI clone, here's a solid stack:

- **Frontend:** React + Tailwind CSS (fast UI iteration, great for mobile-first design)
- **Backend:** Node.js + Express or Next.js API routes
- **AI Integration:** Anthropic Claude API (vision support for food image analysis) or OpenAI GPT-4o
- **Database:** Supabase (Postgres + auth out of the box)
- **Storage:** Supabase Storage or Cloudinary for food images

Recommended flow:
1. User uploads food photo or types description
2. Send to Claude/GPT with a structured prompt asking for nutrition breakdown
3. Parse JSON response → store to DB → update daily dashboard

Start with Next.js so frontend and API routes live together. Want me to scaffold the initial project structure?

---

**[USER]**
Yes, scaffold the project. Use Next.js 14 app router. Also give me the folder structure.

**[AI]**
Here's the recommended folder structure:

```
calorie-tracker/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                  # Dashboard / home
│   ├── log/
│   │   └── page.tsx              # Meal logging page
│   ├── history/
│   │   └── page.tsx              # Past logs
│   └── api/
│       ├── analyze/
│       │   └── route.ts          # AI food analysis endpoint
│       └── meals/
│           └── route.ts          # CRUD for meal logs
├── components/
│   ├── MealCard.tsx
│   ├── NutritionSummary.tsx
│   ├── ImageUploader.tsx
│   └── DailyProgress.tsx
├── lib/
│   ├── supabase.ts
│   └── anthropic.ts             # Claude API helper
├── types/
│   └── meal.ts
└── .env.local
```

I also initialized a `package.json` with these dependencies:
`next`, `react`, `@supabase/supabase-js`, `@anthropic-ai/sdk`, `tailwindcss`, `shadcn/ui`

---

## Session 2 — AI Food Analysis API

**Date:** 2026-06-11

---

**[USER]**
Build the `/api/analyze` route. It should accept either a base64 image or a text description of food, and return calories, protein, carbs, and fat.

**[AI]**
Here's the implementation for `app/api/analyze/route.ts`:

```ts
import Anthropic from "@anthropic-ai/sdk";
import { NextRequest, NextResponse } from "next/server";

const client = new Anthropic();

export async function POST(req: NextRequest) {
  const { image, description } = await req.json();

  const prompt = `You are a nutrition expert. Analyze the food described or shown and return ONLY a JSON object with these fields:
  {
    "food_name": string,
    "calories": number,
    "protein_g": number,
    "carbs_g": number,
    "fat_g": number,
    "serving_size": string,
    "confidence": "high" | "medium" | "low"
  }
  No explanation, just JSON.`;

  const content: any[] = [{ type: "text", text: prompt }];

  if (image) {
    content.push({
      type: "image",
      source: { type: "base64", media_type: "image/jpeg", data: image },
    });
  } else {
    content.push({ type: "text", text: `Food: ${description}` });
  }

  const response = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 500,
    messages: [{ role: "user", content }],
  });

  const raw = response.content[0].type === "text" ? response.content[0].text : "";
  const parsed = JSON.parse(raw.replace(/```json|```/g, "").trim());

  return NextResponse.json(parsed);
}
```

This handles both image and text inputs, returns clean structured nutrition data.

---

**[USER]**
The JSON sometimes comes back with markdown code fences. Fix the parsing so it doesn't break.

**[AI]**
Updated the parse line to strip all possible fence formats:

```ts
const clean = raw
  .replace(/```json\n?/gi, "")
  .replace(/```\n?/g, "")
  .trim();
const parsed = JSON.parse(clean);
```

Also wrapped in try/catch so malformed responses return a 400 with the raw text for debugging:

```ts
try {
  const parsed = JSON.parse(clean);
  return NextResponse.json(parsed);
} catch (e) {
  return NextResponse.json({ error: "Parse failed", raw }, { status: 400 });
}
```

---

## Session 3 — Supabase Schema & Meal Logging

**Date:** 2026-06-11

---

**[USER]**
Design the Supabase schema for storing meal logs.

**[AI]**
Run this SQL in your Supabase SQL editor:

```sql
-- Users are handled by Supabase Auth

create table meals (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users(id) on delete cascade,
  food_name text not null,
  calories int not null,
  protein_g float,
  carbs_g float,
  fat_g float,
  serving_size text,
  meal_type text check (meal_type in ('breakfast', 'lunch', 'dinner', 'snack')),
  image_url text,
  logged_at timestamptz default now(),
  confidence text
);

-- Enable RLS
alter table meals enable row level security;

create policy "Users can only see their own meals"
  on meals for all
  using (auth.uid() = user_id);
```

This ensures users only see their own data via Row Level Security.

---

**[USER]**
Build the `POST /api/meals` route to save a meal after AI analysis.

**[AI]**
```ts
// app/api/meals/route.ts
import { createRouteHandlerClient } from "@supabase/auth-helpers-nextjs";
import { cookies } from "next/headers";
import { NextRequest, NextResponse } from "next/server";

export async function POST(req: NextRequest) {
  const supabase = createRouteHandlerClient({ cookies });
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const body = await req.json();
  const { data, error } = await supabase.from("meals").insert({
    ...body,
    user_id: user.id,
  }).select().single();

  if (error) return NextResponse.json({ error }, { status: 500 });
  return NextResponse.json(data);
}
```

---

## Session 4 — Frontend: Image Uploader + Meal Log UI

**Date:** 2026-06-12

---

**[USER]**
Build the ImageUploader component. Should support drag-and-drop and camera capture on mobile.

**[AI]**
Built `components/ImageUploader.tsx` with:
- Drag-and-drop via `onDragOver` / `onDrop` handlers
- `<input type="file" accept="image/*" capture="environment">` for native camera on mobile
- Preview thumbnail before submission
- Converts file to base64 using `FileReader` before sending to `/api/analyze`
- Loading spinner during analysis

Key snippet:
```tsx
const toBase64 = (file: File): Promise<string> =>
  new Promise((res, rej) => {
    const reader = new FileReader();
    reader.onload = () => res((reader.result as string).split(",")[1]);
    reader.onerror = rej;
    reader.readAsDataURL(file);
  });
```

---

**[USER]**
Build the daily dashboard. Show total calories, protein, carbs, fat for today. Also show a progress bar toward a 2000 calorie goal.

**[AI]**
Built `components/DailyProgress.tsx`:
- Fetches today's meals from Supabase filtered by `logged_at >= today`
- Sums up macros using `.reduce()`
- Renders circular progress ring for calories using SVG
- Color-coded bars: green under goal, yellow at 80%, red over 100%
- Mobile-first layout with sticky header showing calorie count

Also added `components/NutritionSummary.tsx` for the macro breakdown card (protein / carbs / fat in grams with color dots).

---

## Session 5 — Bug Fixes & Polish

**Date:** 2026-06-13

---

**[USER]**
Sometimes the AI returns calories as a string like "350 kcal" instead of a number. Handle this.

**[AI]**
Added a normalizer before saving:

```ts
const normalizeCalories = (val: any): number => {
  if (typeof val === "number") return val;
  return parseInt(String(val).replace(/[^\d]/g, ""), 10) || 0;
};
```

Applied to all numeric fields (`protein_g`, `carbs_g`, `fat_g`) for robustness.

---

**[USER]**
Add a meal type selector (Breakfast / Lunch / Dinner / Snack) to the log page.

**[AI]**
Added a segmented control UI using Tailwind:
```tsx
const MEAL_TYPES = ["breakfast", "lunch", "dinner", "snack"];

<div className="flex gap-2">
  {MEAL_TYPES.map(type => (
    <button
      key={type}
      onClick={() => setMealType(type)}
      className={`capitalize px-3 py-1 rounded-full text-sm font-medium transition
        ${mealType === type ? "bg-green-500 text-white" : "bg-gray-100 text-gray-600"}`}
    >
      {type}
    </button>
  ))}
</div>
```

---

**[USER]**
Add a history page showing past 7 days of meals grouped by date.

**[AI]**
Built `app/history/page.tsx`:
- Queries Supabase for meals from the last 7 days ordered by `logged_at desc`
- Groups by date using `reduce()` into `{ [date]: Meal[] }` map
- Renders each day as an accordion section with total calories shown in the header
- Each meal row shows food name, meal type badge, and calorie count

---

## Session 6 — Deployment

**Date:** 2026-06-14

---

**[USER]**
How do I deploy this to Vercel and connect my Supabase env vars?

**[AI]**
Steps:
1. Push repo to GitHub
2. Import project at vercel.com/new
3. Add environment variables in Vercel dashboard:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `ANTHROPIC_API_KEY`
4. Vercel auto-detects Next.js — hit Deploy
5. In Supabase → Auth → URL Configuration, add your Vercel production URL to allowed redirect URLs

Site is live. AI analysis works end-to-end on mobile.

---

## Summary

| Feature | Status |
|---|---|
| Food photo analysis (Claude Vision) | ✅ Done |
| Text description analysis | ✅ Done |
| Meal logging with Supabase | ✅ Done |
| Daily calorie dashboard | ✅ Done |
| Macro breakdown (protein/carbs/fat) | ✅ Done |
| Meal type tagging | ✅ Done |
| 7-day history view | ✅ Done |
| Mobile camera capture | ✅ Done |
| Deployed on Vercel | ✅ Done |

---

*Log generated from AI-assisted development sessions using Claude (claude-sonnet-4-6)*
