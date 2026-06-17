# AI Calorie Tracker

A mobile app for tracking calories with AI-powered food recognition.

## Features

- AI-powered food scan and calorie estimation
- Meal tracking and history
- User profile and goals
- Premium subscription support via RevenueCat
- Analytics with PostHog
- Error tracking with Sentry

## Tech Stack

- **Framework**: React Native with Expo
- **Navigation**: Expo Router
- **Backend**: Supabase
- **State Management**: TanStack Query
- **UI**: NativeWind (Tailwind CSS)
- **Analytics**: PostHog
- **Error Tracking**: Sentry
- **Payments**: RevenueCat

## Getting Started

1. Clone the repository
2. Copy `.env.example` to `.env` and fill in your API keys
3. Install dependencies: `npm install`
4. Start the app: `npm start`

## Environment Variables

See `.env.example` for required environment variables:

- Supabase URL and key
- RevenueCat API keys
- Sentry DSN
- PostHog key

## Available Scripts

- `npm start` - Start the Expo dev server
- `npm run android` - Run on Android device/emulator
- `npm run ios` - Run on iOS device/emulator
- `npm run web` - Run in web browser
- `npm run typecheck` - Run TypeScript type checking
- `npm test` - Run tests

## Project Structure

```
app/          - Expo Router pages and layouts
components/   - Reusable UI components
hooks/        - Custom React hooks
lib/          - Utilities and services
assets/       - Images and static assets
```

---

## Made by lohith // powered by curiosity.
