# Performance Notes

This document explains the performance considerations and optimizations used in the application.

## 1. Script Loading

- Google Maps JavaScript SDK is loaded **asynchronously** (`async + defer + loading=async`).
- Prevents render-blocking and improves **First Contentful Paint (FCP)**.

## 2. Local Storage Access

Location data saving is initiated when the application is in `idle` state (i.e., the map is stopped).
This then performs a call to save the data using:

- `requestIdleCallback` when available (non-blocking).
  - `requestIdleCallback` is a browser API that lets you schedule work to run when the browser is idle (i.e., when it’s not busy doing critical tasks like rendering, animations, or user interactions). It’s similar to setTimeout, but smarter. Instead of running after a fixed time, the callback is run by the browser during a free moment.

- Fallback to debounced `setTimeout` if `requestIdleCallback` is undefined for some reason.

## 3. Map Rendering

- Disabled unnecessary controls:
  - `fullscreenControl`, `mapTypeControl`, `streetViewControl`.
- Enabled `gestureHandling: greedy` for smoother mobile experience.
- Optimized pin rendering with lightweight centered SVG overlay instead of heavy markers.

## 4. Reverse Geocoding

- Reverse geocoding requests only fire when the map is idle, reducing API load and UI lag.

## 5. Asset Optimization

- Custom SVG pointer instead of bitmap reduces load time and memory usage.
- CSS uses `will-change: transform` on map container for smoother transitions.

## 6. Responsiveness

- Map container uses `width: 100vw; height: 100vh;` to fully utilize the viewport.
- Responsive overlays adjust padding and font size on smaller devices.

## 7. Monitoring

- Console warnings are logged when:
  - Google Maps API key is missing.
  - LocalStorage read/write fails.
- Makes debugging performance issues easier.
