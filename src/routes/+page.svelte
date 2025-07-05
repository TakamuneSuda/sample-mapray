<script lang="ts">
	import { onMount } from 'svelte';
	import mapray from '@mapray/mapray-js';
	import maprayui from '@mapray/ui';
	import '@mapray/ui/mapray.css';
	import GeoJSONLayer from '$lib/components/GeoJSONLayer.svelte';
	import ColorControl from '$lib/components/ColorControl.svelte';

	// Reference to the DOM element for the Mapray viewer container
	let mapContainer: HTMLElement;

	// Instance of the Mapray viewer
	let stdViewer: maprayui.StandardUIViewer | null = $state(null);

	let renderMode: 'SURFACE' | 'WIREFRAME' | 'HIDE' = $state('SURFACE');

	let showContour: boolean = $state(false);
	let contourLayer: any = null;

	// Point cloud dataset management
	const pointCloudDatasets = [
		{
			id: '5153900512411648',
			name: '富士山',
			camera: {
				longitude: 138.73035,
				latitude: 35.224604,
				height: 8000,
				pitch: 70,
				yaw: 0
			}
		},
		{
			id: '5120507091353600',
			name: '河津七滝ループ橋',
			camera: {
				longitude: 138.94204,
				latitude: 34.78731,
				height: 475,
				pitch: 65,
				yaw: 25
			}
		},
		{
			id: '5107296979910656',
			name: 'ALB測量',
			camera: {
				longitude: 131.153185,
				latitude: 33.420539,
				height: 230,
				pitch: 65,
				yaw: 5
			}
		}
	];

	let selectedDatasetId = $state('5153900512411648');
	let currentPointCloud: any = null;

	// Update render mode when renderMode changes
	$effect(() => {
		if (stdViewer) {
			switch (renderMode) {
				case 'SURFACE':
					stdViewer.viewer.setVisibility(mapray.Viewer.Category.GROUND, true);
					stdViewer.viewer.render_mode = mapray.Viewer.RenderMode.SURFACE;
					break;
				case 'WIREFRAME':
					stdViewer.viewer.setVisibility(mapray.Viewer.Category.GROUND, true);
					stdViewer.viewer.render_mode = mapray.Viewer.RenderMode.WIREFRAME;
					break;
				case 'HIDE':
					stdViewer?.viewer.setVisibility(mapray.Viewer.Category.GROUND, false);
			}
		}
	});

	// Update point cloud when dataset selection changes
	$effect(() => {
		if (stdViewer && selectedDatasetId) {
			// Remove current point cloud if exists
			if (currentPointCloud) {
				stdViewer.viewer.point_cloud_collection.remove(currentPointCloud);
				currentPointCloud = null;
			}

			// Find selected dataset
			const selectedDataset = pointCloudDatasets.find(
				(dataset) => dataset.id === selectedDatasetId
			);

			if (selectedDataset) {
				// 位置設定
				stdViewer.setCameraPosition(selectedDataset.camera);

				// 向きを手動で設定する場合はsetLookAtPositionを使わない
				stdViewer.setCameraAngle({
					roll: 0,
					pitch: selectedDataset.camera.pitch ?? 45, // 下向き45度
					yaw: selectedDataset.camera.yaw ?? 0 // 南向き（lookAtの方向に応じて調整）
				});

				// Point cloud追加
				const cloudApi = new mapray.cloud.CloudApiV2({
					tokenType: mapray.cloud.CloudApi.TokenType.API_KEY,
					token: ACCESS_TOKEN
					// userIdは削除
				});

				const resource = cloudApi.getPointCloudDatasetAsResource(selectedDatasetId);
				const pointCloudProvider = new mapray.StandardPointCloudProvider(resource);
				currentPointCloud = stdViewer.viewer.point_cloud_collection.add(pointCloudProvider);
			}
		}
	});

	// // Update contour visibility when showContour changes
	// $effect(() => {
	// 	if (stdViewer) {
	// 		for (let i = 0; i < stdViewer.viewer.layers.length(); i++) {
	// 			const layer = stdViewer.viewer.layers.get(i);
	// 			if (layer.type === mapray.Layer.Type.CONTOUR) {
	// 				layer.visibility = showContour;
	// 				break;
	// 			}
	// 		}
	// 	}
	// });

	// Specify the pre-obtained access token. Reads from environment variable.
	const ACCESS_TOKEN = import.meta.env.VITE_MAPRAY_ACCESS_TOKEN;

	const USER_ID = import.meta.env.VITE_MAPRAY_USER_ID;

	const geojsonPointUrl = 'sample/geojson/points.geojson';
	const geojsonLineUrl = 'sample/geojson/lines.geojson';
	const geojsonPolygonUrl = 'sample/geojson/polygons.geojson';

	// // Reactive state for GeoJSON styling options using Svelte 5 $state
	let colors = $state({
		lineColor: [1, 0, 0], // Default blue
		fillColor: [0, 0, 1], // Default red
		pointFGColor: [0, 1, 0], // Default green
		pointBGColor: [1, 1, 0] // Default yellow
	});

	let option = $derived(() => ({
		getAltitudeMode: () => mapray.AltitudeMode.CLAMP,
		getLineColor: () => colors.lineColor,
		getFillColor: () => colors.fillColor,
		getPointFGColor: () => colors.pointFGColor,
		getPointBGColor: () => colors.pointBGColor,
		getLineWidth: () => 3
	}));

	// let option = $state({
	// 	getAltitudeMode: () => mapray.AltitudeMode.CLAMP,
	// 	lineColor: [0, 0, 1], // Default blue
	// 	fillColor: [1, 0, 0], // Default red
	// 	pointFGColor: [0, 1, 0], // Default green
	// 	pointBGColor: [1, 1, 0], // Default yellow
	// });

	// let option = $derived(() => ({
	// 	getAltitudeMode: () => mapray.AltitudeMode.CLAMP,
	// 	getLineColor: () => colors.lineColor,
	// 	getFillColor: () => colors.fillColor,
	// 	getPointFGColor: () => colors.pointFGColor,
	// 	getPointBGColor: () => colors.pointBGColor,
	// 	getLineWidth: () => 3
	// }));

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

		stdViewer.viewer.logo_controller.setVisibility(true);
		stdViewer.viewer.logo_controller.setPosition(
			mapray.LogoController.ContainerPosition.BOTTOM_LEFT
		);
		stdViewer.viewer.logo_controller.setCompact(false);

		// Point cloud will be handled by the $effect above

		// Position of Initial Point
		const initialPosition = new mapray.GeoPoint(138.72078, 35.2306823, 6530);

		// Change camera position with pitch
		stdViewer.setCameraPosition({
			longitude: initialPosition.longitude,
			latitude: initialPosition.latitude,
			height: initialPosition.altitude
		});

		// Change look-at position
		// stdViewer.setLookAtPosition({
		// 	longitude: initialPosition.longitude,
		// 	latitude: initialPosition.latitude,
		// 	height: initialPosition.altitude + 3000
		// });

		// Set camera pitch to look down at 45 degrees
		// stdViewer.setCameraDirection({
		// 	pitch: 0,  // Look down at 45 degrees
		// 	yaw: 130       // Face north
		// });
		stdViewer.setCameraAngle({
			roll: 0, // No roll
			pitch: 60, // Look down at 45 degrees
			yaw: 0 // Face north
		});

		// Create and add pin entity
		const pin = new mapray.PinEntity(stdViewer.viewer.scene);
		pin.altitude_mode = mapray.AltitudeMode.RELATIVE;
		// pin.addMakiIconPin('mountain-15', mtFujiPosition);
		stdViewer.addEntity(pin);

		// Return a cleanup function to be called when the component is destroyed
		return () => {
			if (stdViewer) {
				stdViewer.destroy();
				stdViewer = null;
			}
		};
	});

	// // レイヤーの初期化（一度のみ）
	// $effect(() => {
	// 	if (!stdViewer || contourLayer) return;

	// 	const initContourLayer = async () => {
	// 		try {
	// 			contourLayer = await stdViewer.addLayer({
	// 				type: mapray.Layer.Type.CONTOUR,
	// 				visibility: true,
	// 				opacity: 0.7,
	// 				interval: 150,
	// 				line_width: 2,
	// 				color: [1, 0, 0, 1.0]
	// 			});
	// 			console.log('等高線レイヤーを初期化しました:', contourLayer);
	// 		} catch (error) {
	// 			console.error('等高線レイヤーの初期化に失敗:', error);
	// 		}
	// 	};

	// 	initContourLayer();
	// });

	// 表示状態の切り替え（レイヤーを削除しない）
	$effect(() => {
		if (contourLayer) {
			try {
				contourLayer.visibility = showContour;
				console.log('等高線レイヤーの表示状態を変更:', showContour);
			} catch (error) {
				console.error('等高線レイヤーの表示状態変更に失敗:', error);
			}
		}
	});
</script>

<div
	bind:this={mapContainer}
	class="relative h-screen w-screen"
	role="application"
	aria-label="Map viewer with GeoJSON file drop area"
>
	{#if stdViewer}
		<!-- Mapray viewer will be rendered here -->
		<!-- <GeoJSONLayer url={geojsonPointUrl} scene={stdViewer.viewer.scene} options={option} />
		<GeoJSONLayer url={geojsonLineUrl} scene={stdViewer.viewer.scene} options={option} />
		<GeoJSONLayer url={geojsonPolygonUrl} scene={stdViewer.viewer.scene} options={option} /> -->
		<!-- Control panel -->
		<div class="control-panel">
			<h3>Map Controls</h3>

			<!-- Render Mode Toggle -->
			<div class="render-mode-control">
				<label>Render Mode:</label>
				<select bind:value={renderMode}>
					<option value="SURFACE">Surface</option>
					<option value="WIREFRAME">Wireframe</option>
					<option value="HIDE">Hide</option>
				</select>
			</div>

			<!-- Point Cloud Dataset Selector -->
			<div class="dataset-control">
				<label>Point Cloud Dataset:</label>
				<select bind:value={selectedDatasetId}>
					{#each pointCloudDatasets as dataset}
						<option value={dataset.id}>{dataset.name}</option>
					{/each}
				</select>
			</div>

			<!-- <div>
				<label for="contour-visibility">Show Contours:</label>
				<input type="checkbox" id="contour-visibility" bind:checked={showContour} />
			</div> -->

			<!-- <h4>GeoJSON Styling</h4>
			<ColorControl label="Line Color" bind:color={colors.lineColor} />
			<ColorControl label="Fill Color" bind:color={colors.fillColor} />
			<ColorControl label="Point Foreground" bind:color={colors.pointFGColor} />
			<ColorControl label="Point Background" bind:color={colors.pointBGColor} /> -->
		</div>
	{/if}
</div>

<style>
	.control-panel {
		position: absolute;
		top: 20px;
		left: 20px;
		background: rgba(255, 255, 255, 0.95);
		padding: 16px;
		border-radius: 8px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
		max-width: 280px;
		z-index: 1000;
	}

	.control-panel h3 {
		margin: 0 0 12px 0;
		font-size: 16px;
		font-weight: 600;
		color: #333;
	}

	.control-panel h4 {
		margin: 16px 0 8px 0;
		font-size: 14px;
		font-weight: 600;
		color: #555;
	}

	.render-mode-control,
	.dataset-control {
		margin-bottom: 16px;
	}

	.render-mode-control label,
	.dataset-control label {
		display: block;
		margin-bottom: 4px;
		font-size: 14px;
		font-weight: 500;
		color: #333;
	}

	.render-mode-control select,
	.dataset-control select {
		width: 100%;
		padding: 6px 8px;
		border: 1px solid #ddd;
		border-radius: 4px;
		font-size: 14px;
		background: white;
	}

	.control-panel :global(.color-control) {
		margin-bottom: 12px;
	}

	.control-panel :global(.color-control:last-child) {
		margin-bottom: 0;
	}
</style>
