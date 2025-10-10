<script lang="ts">
	import { onMount } from 'svelte';
	import mapray from '@mapray/mapray-js';
	import maprayui from '@mapray/ui';

	// Reference to the DOM element for the Mapray viewer container
	let mapContainer: HTMLElement;

	// Instance of the Mapray viewer
	let stdViewer: maprayui.StandardUIViewer | null = null;

	// Specify the pre-obtained access token. Reads from environment variable.
	const ACCESS_TOKEN = import.meta.env.VITE_MAPRAY_ACCESS_TOKEN;

	// カメラの回転角度
	type RollPitchYawData = {
		roll: number;
		pitch: number;
		yaw: number;
	};

	let cameraRollPitchYaw = $state<RollPitchYawData>({
		roll: 0,
		pitch: 0,
		yaw: 0
	});

	// Run after the component is mounted
	onMount(() => {
		if (!mapContainer) {
			console.error('Map container element not found');
			return;
		}
		if (!ACCESS_TOKEN) {
			console.error('Mapray access token is not defined');
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
		pin.addMakiIconPin('mountain-15', mtFujiPosition);
		stdViewer.addEntity(pin);

		const viewer = stdViewer as any;


		// Return a cleanup function to be called when the component is destroyed
		return () => {
			if (stdViewer) {
				stdViewer.destroy(); // Release viewer resources
				stdViewer = null;
			}
		};
	});

	// カメラの座標と注視点の型定義
	type CameraPosition = {
		position: { longitude: number; latitude: number; height: number };
		lookAt: { longitude: number; latitude: number; height: number };
	};

	// カメラ位置の座標
	const cameraPositions: {
		[key: string]: CameraPosition;
	} = {
		fuji: {
			position: {
				longitude: 138.746078,
				latitude: 35.388442,
				height: 6000
			},
			lookAt: {
				longitude: 138.730034,
				latitude: 35.361835,
				height: 3600
			}
		},
		tokyo: {
			position: {
				longitude: 139.764728,
				latitude: 35.67141,
				height: 1000
			},
			lookAt: {
				longitude: 139.767206,
				latitude: 35.680806,
				height: 100
			}
		}
	};

	// カメラ位置の更新
	const changeCameraPosition = (pos: CameraPosition) => {
		if (stdViewer) {
			stdViewer.setCameraPosition({
				longitude: pos.position.longitude,
				latitude: pos.position.latitude,
				height: pos.position.height
			});
			stdViewer.setLookAtPosition({
				longitude: pos.lookAt.longitude,
				latitude: pos.lookAt.latitude,
				height: pos.lookAt.height
			});
		}
	};

</script>

<div class="relative h-[100vh] w-full">
	<div bind:this={mapContainer} class="absolute top-0 left-0 h-full w-full"></div>
	<div class="bg-opacity-75 absolute top-4 left-4 rounded bg-white p-2 shadow">
		<div class="flex items-center gap-2">
			<p>camera position</p>
			<select
				class="rounded border p-1"
				onchange={(e) => changeCameraPosition(cameraPositions[e.target.value])}
			>
				{#each Object.entries(cameraPositions) as [key, pos]}
					<option value={key}>{key}</option>
				{/each}
			</select>
		</div>
		<div>
			<div>
				<p>roll: {cameraRollPitchYaw.roll}°</p>
				<input type="range" min="-180" max="180" bind:value={cameraRollPitchYaw.roll} />
			</div>
			<div>
				<p>pitch: {cameraRollPitchYaw.pitch}°</p>
				<input type="range" min="-180" max="180" bind:value={cameraRollPitchYaw.pitch} />
			</div>
			<div>
				<p>yaw: {cameraRollPitchYaw.yaw}°</p>
				<input type="range" min="-180" max="180" bind:value={cameraRollPitchYaw.yaw} />
			</div>
		</div>
	</div>
</div>
