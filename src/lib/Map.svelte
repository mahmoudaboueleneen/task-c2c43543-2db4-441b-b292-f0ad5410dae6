<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { browser } from '$app/environment';

  /**
   * Type alias for Google Maps LatLngLiteral.
   */
  type LatLngLiteral = google.maps.LatLngLiteral;

  /**
   * Key for storing user location in localStorage.
   */
  const STORAGE_KEY = 'delivery_map_user_location_v1';
  /**
   * Fallback center (Cairo, Egypt) if geolocation and saved location are unavailable.
   */
  const fallbackCenter: LatLngLiteral = { lat: 30.0444, lng: 31.2357 };
  /**
   * Initial map zoom level.
   */
  const initialZoom = 15;
  /**
   * Debounce time in milliseconds for saving location.
   */
  const DEBOUNCE_MS = 700;

  /**
   * Reference to the map container element.
   */
  let mapEl: HTMLDivElement | null = null;
  /**
   * Google Maps Map instance.
   */
  let map: google.maps.Map | null = null;

  /**
   * Last saved location string to avoid redundant writes.
   */
  let lastSaved: string | null = null;
  /**
   * Timer for debounced saving.
   */
  let saveTimer: ReturnType<typeof setTimeout> | null = null;

  /**
   * Human-readable address for bottom card.
   */
  let address: string = 'Loading…';

  /**
   * Save location to localStorage asynchronously, debounced.
   * @param coords Coordinates to save
   */
  function saveLocationAsync(coords: LatLngLiteral): void {
    const str = JSON.stringify(coords);
    if (str === lastSaved) return;
    lastSaved = str;

    if (typeof requestIdleCallback !== 'undefined') {
      requestIdleCallback(() => {
        try {
          localStorage.setItem(STORAGE_KEY, str);
        } catch (e) {
          console.warn('Failed to write location', e);
        }
      });
    } else {
      if (saveTimer) clearTimeout(saveTimer);
      saveTimer = setTimeout(() => {
        try {
          localStorage.setItem(STORAGE_KEY, str);
        } catch (e) {
          console.warn('Failed to write location', e);
        }
      }, DEBOUNCE_MS);
    }
  }

  /**
   * Load saved location from localStorage.
   * @returns Coordinates or null if not found
   */
  function loadSavedLocation(): LatLngLiteral | null {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      return raw ? (JSON.parse(raw) as LatLngLiteral) : null;
    } catch (e) {
      console.warn('Failed to read saved location', e);
      return null;
    }
  }

  /**
   * Dynamically load the Google Maps script.
   * @param apiKey The Google Maps API key
   * @returns Promise that resolves when the script is loaded
   */
  function loadGoogleMapsScript(apiKey: string): Promise<typeof google> {
    return new Promise((resolve, reject) => {
      if (window.google && window.google.maps) {
        resolve(window.google);
        return;
      }

      const url = `https://maps.googleapis.com/maps/api/js?key=${apiKey}&v=weekly&libraries=places`;
      const s = document.createElement('script');
      s.src = url;
      s.async = true;
      s.defer = true;
      s.setAttribute('loading', 'async');

      s.onload = () => {
        if (window.google && window.google.maps) {
          resolve(window.google);
        } else {
          reject(new Error('Google Maps failed to initialize'));
        }
      };

      s.onerror = (err) => reject(err);
      document.head.appendChild(s);
    });
  }

  /**
   * Reverse geocode coordinates into a street address.
   */
  async function reverseGeocode(coords: LatLngLiteral): Promise<string> {
    return new Promise((resolve, reject) => {
      const geocoder = new google.maps.Geocoder();
      geocoder.geocode({ location: coords }, (results, status) => {
        if (status === 'OK' && results && results[0]) {
          resolve(results[0].formatted_address);
        } else {
          console.warn('Geocoder failed due to:', status);
          resolve('Unknown location');
        }
      });
    });
  }

  /**
   * Try to get the user's geolocation.
   * Falls back to Cairo if denied/unavailable.
   */
  async function getInitialCenter(): Promise<LatLngLiteral> {
    const saved = loadSavedLocation();
    if (saved) return saved;

    if (browser && 'geolocation' in navigator) {
      return new Promise((resolve) => {
        navigator.geolocation.getCurrentPosition(
          (pos) => {
            resolve({
              lat: pos.coords.latitude,
              lng: pos.coords.longitude,
            });
          },
          (err) => {
            console.warn('Geolocation failed:', err.message);
            resolve(fallbackCenter);
          },
          { enableHighAccuracy: true, timeout: 5000 }
        );
      });
    }

    return fallbackCenter;
  }

  /**
   * Lifecycle hook to initialize the map when component mounts.
   */
  onMount(async () => {
    if (!browser) return;

    const apiKey = import.meta.env.VITE_GOOGLE_MAPS_API_KEY;
    if (!apiKey) {
      console.warn('VITE_GOOGLE_MAPS_API_KEY is not set. Map will not load.');
      return;
    }

    try {
      await loadGoogleMapsScript(apiKey);
    } catch (err) {
      console.error('Failed to load Google Maps script', err);
      return;
    }

    const startCenter = await getInitialCenter();

    map = new google.maps.Map(mapEl as HTMLDivElement, {
      center: startCenter,          // initial center of the map (LatLngLiteral),
                                    // comes from saved location, geolocation, or fallback Cairo.

      zoom: initialZoom,            // initial zoom level (integer). Higher value = closer view.
                                    // 15 ≈ neighborhood/street level.

      fullscreenControl: false,     // hides the fullscreen toggle button (saves UI space & reduces clutter).

      mapTypeControl: false,        // hides the map type selector (e.g., Roadmap, Satellite).
                                    // avoids extra UI elements not needed for the app.

      streetViewControl: false,     // hides the Street View button.
                                    // keeps focus on delivery location, reduces distraction.

      clickableIcons: false,        // prevents POI (point of interest) icons from being interactive.
                                    // avoids accidental taps on shops, landmarks, etc.

      gestureHandling: 'greedy',    // all touch gestures (scroll, pinch, drag) pan/zoom the map.
                                    // better UX on mobile, prevents conflicts with page scrolling.
    });

    // initial reverse geocode to get the actual street address,
    // to be displayed in the bottom card.
    address = await reverseGeocode(startCenter);

    // on idle (i.e., when the user stops interacting with the map),
    // perform a call to save the coords and update address in the background asynchronously.
    map.addListener('idle', async () => {
      const center = map!.getCenter();
      if (center) {
        const coords = { lat: center.lat(), lng: center.lng() };
        saveLocationAsync(coords);
        address = await reverseGeocode(coords);
      }
    });
  });

  /**
   * Lifecycle hook to perform cleanup when component is destroyed
   * and to clear any pending save timers.
   */
  onDestroy(() => {
    if (saveTimer) clearTimeout(saveTimer);
  });
</script>

<div class="map-wrapper">
  <div bind:this={mapEl} class="map-el" aria-label="Map"></div>

  <!-- Custom centered pin instead of the default marker -->
  <!-- This helps with performance as we don't need to re-render markers -->
  <!-- Also allows for a cleaner UI -->
  <div class="map-pointer">
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24">
      <path
        d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7z"
        fill="#007aff"
      />
      <circle cx="12" cy="9" r="2.5" fill="white" />
    </svg>
  </div>

  <!-- floating header -->
  <div class="top-bar">
    <button class="back-btn">←</button>
    <h2>Confirm Delivery Location</h2>
  </div>

  <!-- floating bottom card -->
  <div class="bottom-card">
    <p class="address-label">Delivery Location:</p>
    <p class="address-value">{address}</p>
    <button class="confirm-btn">Confirm Location</button>
  </div>
</div>

<style>
  .map-wrapper {
    position: relative;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    font-family: system-ui, sans-serif;
  }
  .map-el {
    width: 100%;
    height: 100%;
    will-change: transform;
  }

  /* Pin */
  .map-pointer {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -100%);
    width: 40px;
    height: 40px;
    z-index: 10;
    pointer-events: none;
  }
  .map-pointer svg {
    width: 100%;
    height: 100%;
    display: block;
    filter: drop-shadow(0 1px 1px rgba(0, 0, 0, 0.3));
  }

  /* Top Bar */
  .top-bar {
    position: absolute;
    top: 16px;
    left: 16px;
    right: 16px;
    display: flex;
    align-items: center;
    gap: 12px;
    background: rgba(255, 255, 255, 0.95);
    padding: 10px 14px;
    border-radius: 12px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    z-index: 20;
  }
  .top-bar h2 {
    font-size: 15px;
    margin: 0;
    font-weight: 600;
    color: #333;
  }
  .back-btn {
    background: none;
    border: none;
    font-size: 20px;
    cursor: pointer;
    color: #007aff;
  }

  /* Bottom Card */
  .bottom-card {
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    background: #fff;
    padding: 16px;
    border-top-left-radius: 16px;
    border-top-right-radius: 16px;
    box-shadow: 0 -2px 12px rgba(0, 0, 0, 0.15);
    z-index: 20;
  }
  .address-label {
    font-size: 12px;
    color: #777;
    margin: 0 0 4px;
  }
  .address-value {
    font-size: 14px;
    font-weight: 500;
    margin: 0 0 12px;
  }
  .confirm-btn {
    width: 100%;
    background: #007aff;
    color: #fff;
    border: none;
    padding: 12px;
    border-radius: 10px;
    font-size: 15px;
    font-weight: 600;
    cursor: pointer;
  }
  .confirm-btn:hover {
    background: #0063cc;
  }
</style>
