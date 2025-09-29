# Strategies Used in Implementation

This document outlines the strategies and design decisions behind the application.

## 1. Map SDK Choice

- **Google Maps JavaScript SDK** was selected for reliable map rendering and geocoding features.

## 2. State Persistence

- The selected location is saved in `localStorage` to preserve user choices between sessions.
- Debounced saving ensures writes do not block UI performance.
- Initial map location defaults to the user's current location if the user accepts to share his location on initial load and if there is no saved location in `localStorage`. Otherwise it defaults to Cairo, Egypt.

## 3. Performance & UX

- Used `requestIdleCallback` for background saves when supported.
- Applied `debounce` fallback with configurable delay to minimize blocking main thread.
- Applied lazy loading (`loading=async`) for Google Maps script to avoid render-blocking.

## 4. User Interaction

- Users can drag the map around to center the pin over a selected location.
- The pin is always centered for clarity.
- Reverse geocoding shows the real-world address of the chosen point instead of just the coordinates.

## 5. Code Documentation

- Inline code comments explain the purpose of each variable, function, and lifecycle hook.

## 6. Accessibility

- UI overlays provide clear instructions.

## 7. Styling & UI

- Overlay card at the bottom displays the current street address dynamically.
- Styled with subtle shadows, rounded corners, and responsive font sizes for mobile users.
- A custom location pin is used to match the color and styling of the application.
