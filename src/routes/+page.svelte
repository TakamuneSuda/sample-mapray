<script lang="ts">
  import { onMount } from 'svelte';
  import mapray from "@mapray/mapray-js";
  import maprayui from "@mapray/ui";

  // Reference to the DOM element for the Mapray viewer container
  let mapContainer: HTMLElement;

  // Instance of the Mapray viewer
  let stdViewer: maprayui.StandardUIViewer | null = null;

  // Specify the pre-obtained access token. Reads from environment variable.
  const ACCESS_TOKEN = import.meta.env.VITE_MAPRAY_ACCESS_TOKEN;

  // Run after the component is mounted
  onMount(() => {
    if (!mapContainer) {
      console.error("Map container element not found");
      return;
    }
    if (!ACCESS_TOKEN) {
      console.error("Mapray access token is not defined");
      // Optionally, display an error message to the user here
      return;
    }

    // Initialize Mapray viewer
    stdViewer = new maprayui.StandardUIViewer(mapContainer, ACCESS_TOKEN);

    // Position of Mt. Fuji
    const mtFujiPosition = new mapray.GeoPoint(138.72884, 35.36423, 0);

    // Change camera position
    stdViewer.setCameraPosition({
      longitude: mtFujiPosition.longitude,
      latitude: mtFujiPosition.latitude - 0.05,
      height: mtFujiPosition.altitude + 6000
    });

    // Change look-at position
    stdViewer.setLookAtPosition({
      longitude: mtFujiPosition.longitude,
      latitude: mtFujiPosition.latitude,
      height: mtFujiPosition.altitude + 3000
    });

    // Create and add pin entity
    const pin = new mapray.PinEntity(stdViewer.viewer.scene);
    pin.altitude_mode = mapray.AltitudeMode.RELATIVE;
    pin.addMakiIconPin( "mountain-15", mtFujiPosition );
    stdViewer.addEntity(pin);

    // Return a cleanup function to be called when the component is destroyed
    return () => {
      if (stdViewer) {
        stdViewer.destroy(); // Release viewer resources
        stdViewer = null;
      }
    };
  });

</script>

<div
  bind:this={mapContainer}
  class="w-screen h-screen"
>
</div>