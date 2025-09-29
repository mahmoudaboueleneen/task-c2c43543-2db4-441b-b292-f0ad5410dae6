# Installation Guide

This document provides setup instructions for the application.

## 1. Prerequisites

- [Node.js](https://nodejs.org/) (v20 or later)
- [npm](https://www.npmjs.com/) (comes with Node.js)
- A valid **Google Maps API Key**
  - You can create one at the [Google Cloud Console](https://console.cloud.google.com/).
  - In 'Application restrictions', select 'Websites'.
  - In 'Website restrictions', add `http://localhost:5173/*` (the local dev url) and `http://localhost:4173/*` for build preview. For production, add the production url as well.
  - In 'API restrictions', only select 'Maps JavaScript API' (for map loading) and 'Geocoding API' (for translating coordinates to street address)

## 2. Clone the Repository

```bash
git clone https://github.com/mahmoudaboueleneen/task-c2c43543-2db4-441b-b292-f0ad5410dae6.git
cd task-c2c43543-2db4-441b-b292-f0ad5410dae6
```

## 3. Install Dependencies

```bash
npm install
```

## 4. Configure Environment Variables

Create a .env file in the project root and add:

```bash
VITE_GOOGLE_MAPS_API_KEY= # your Google Maps API key
```

## 5. Run Development Server

```bash
npm run dev
```

## 6. Build for Production

```bash
npm run build
npm run preview
```
