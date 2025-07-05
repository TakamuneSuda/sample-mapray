<script lang="ts">
	import { onMount } from 'svelte';
	import mapray from '@mapray/mapray-js';
	import maprayui from '@mapray/ui';
	import GeoJSONLayer from '$lib/components/GeoJSONLayer.svelte';
	import ColorControl from '$lib/components/ColorControl.svelte';

	// Reference to the DOM element for the Mapray viewer container
	let mapContainer: HTMLElement;

	// Custom RealtimeViewer class
	class RealtimeViewer extends maprayui.StandardUIViewer {
		cameraListeners: Array<(cameraInfo: any) => void>;

		constructor(container, accessToken) {
			super(container, accessToken);
			this.cameraListeners = [];
		}

		// カメラ変更リスナーを追加
		addCameraListener(callback) {
			this.cameraListeners.push(callback);
		}

		onUpdateFrame(delta_time) {
			super.onUpdateFrame(delta_time);

			const position = this.getCameraPosition();
			const direction = this.getCameraDirection();

			const cameraInfo = {
				...position,
				yaw: direction.yaw,
				pitch: direction.pitch
			};

			// 全リスナーに通知
			this.cameraListeners.forEach(callback => {
				callback(cameraInfo);
			});
		}

		getCameraDirection() {
			const camera = this.viewer.camera;
			const matrix = camera.view_to_gocs;
			
			if (!matrix) {
				return { yaw: 0, pitch: 0 };
			}
			
			// Extract rotation matrix components
			// Matrix is 4x4, we need the 3x3 rotation part
			const r11 = matrix[0], r12 = matrix[1], r13 = matrix[2];
			const r21 = matrix[4], r22 = matrix[5], r23 = matrix[6];
			const r31 = matrix[8], r32 = matrix[9], r33 = matrix[10];
			
			// Alternative approach: YXZ rotation order (Yaw-Pitch-Roll)
			// This should make pitch more independent from yaw
			let pitch, yaw;
			
			// Calculate yaw first (rotation around Y-axis)
			// Use horizontal projection for true azimuth
			yaw = Math.atan2(r31, r11) * (180 / Math.PI);
			if (yaw < 0) yaw += 360;
			
			// Calculate pitch (rotation around X-axis after yaw)
			// Use the Z-component of the forward vector after yaw rotation
			const cosPitch = Math.sqrt(r31 * r31 + r11 * r11);
			if (cosPitch > 1e-6) {
				pitch = Math.atan2(-r21, cosPitch) * (180 / Math.PI);
			} else {
				// Gimbal lock: looking straight up or down
				pitch = Math.atan2(-r21, r22) * (180 / Math.PI);
			}
			
			console.log('Matrix rotation components:', {
				r11, r12, r13,
				r21, r22, r23,
				r31, r32, r33
			});
			console.log('YXZ order - Pitch:', pitch.toFixed(1), 'Yaw:', yaw.toFixed(1));
			console.log('CosPitch:', cosPitch.toFixed(3));
			
			return { yaw, pitch };
		}
	}

	// Instance of the Mapray viewer
	let stdViewer: RealtimeViewer | null = $state(null);

	let renderMode: 'SURFACE' | 'WIREFRAME' | 'HIDE' = $state('SURFACE');
	
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
			},
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
		},
	];
	
	let selectedDatasetId = $state('5153900512411648');
	let currentPointCloud: any = null;
	
	// Real-time camera position tracking
	let cameraPosition = $state({
		longitude: 0,
		latitude: 0,
		height: 0,
		pitch: 0,
		yaw: 0
	});
	
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
			const selectedDataset = pointCloudDatasets.find(dataset => dataset.id === selectedDatasetId);
			
			if (selectedDataset) {
				// 位置設定
				stdViewer.setCameraPosition(selectedDataset.camera);
				
				// 向きを手動で設定する場合はsetLookAtPositionを使わない
				stdViewer.setCameraAngle({
					roll: 0,
					pitch: selectedDataset.camera.pitch ?? 45,  // 下向き45度
					yaw: selectedDataset.camera.yaw ?? 0    // 南向き（lookAtの方向に応じて調整）
				});
				
				// Point cloud追加
				const cloudApi = new mapray.cloud.CloudApiV2({
					tokenType: mapray.cloud.CloudApi.TokenType.API_KEY,
					token: ACCESS_TOKEN,
					// userIdは削除
				});

				const resource = cloudApi.getPointCloudDatasetAsResource(selectedDatasetId);
				const pointCloudProvider = new mapray.StandardPointCloudProvider(resource);
				currentPointCloud = stdViewer.viewer.point_cloud_collection.add(pointCloudProvider);
			}
		}
	});

	// Specify the pre-obtained access token. Reads from environment variable.
	const ACCESS_TOKEN = import.meta.env.VITE_MAPRAY_ACCESS_TOKEN;

	const USER_ID = import.meta.env.VITE_MAPRAY_USER_ID;

	const geojsonPointUrl = 'sample/geojson/points.geojson';
	const geojsonLineUrl = 'sample/geojson/lines.geojson';
	const geojsonPolygonUrl = 'sample/geojson/polygons.geojson';

	// // Reactive state for GeoJSON styling options using Svelte 5 $state
	let colors = $state({
		lineColor: [0, 0, 1], // Default blue
		fillColor: [1, 0, 0], // Default red
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
		stdViewer = new RealtimeViewer(mapContainer, ACCESS_TOKEN);
		
		// Add camera listener for real-time position tracking
		stdViewer.addCameraListener((position) => {
			cameraPosition = {
				longitude: Number(position.longitude.toFixed(6)),
				latitude: Number(position.latitude.toFixed(6)),
				height: Number(position.height.toFixed(1)),
				pitch: Number(position.pitch.toFixed(1)),
				yaw: Number(position.yaw.toFixed(1))
			};
			console.log('Camera position updated:', cameraPosition);
		});
		

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
	})

		// Create and add pin entity
		const pin = new mapray.PinEntity(stdViewer.viewer.scene);
		pin.altitude_mode = mapray.AltitudeMode.RELATIVE;
		pin.addMakiIconPin('mountain-15', mtFujiPosition);
		stdViewer.addEntity(pin);
	
		// Return a cleanup function to be called when the component is destroyed
		return () => {
			if (stdViewer) {
				stdViewer.destroy();
				stdViewer = null;
			}
		};
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
		<GeoJSONLayer url={geojsonPointUrl} scene={stdViewer.viewer.scene} options={option} />
		<GeoJSONLayer url={geojsonLineUrl} scene={stdViewer.viewer.scene} options={option} />
		<GeoJSONLayer url={geojsonPolygonUrl} scene={stdViewer.viewer.scene} options={option} />
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

			<!-- Real-time Camera Position Display -->
			<div class="camera-info">
				<h4>Camera Position</h4>
				<div class="camera-coords">
					<div>経度: {cameraPosition.longitude}°</div>
					<div>緯度: {cameraPosition.latitude}°</div>
					<div>高度: {cameraPosition.height}m</div>
					<div>Pitch: {cameraPosition.pitch}°</div>
					<div>Yaw: {cameraPosition.yaw}°</div>
				</div>
			</div>

			<h4>GeoJSON Styling</h4>
			<ColorControl label="Line Color" bind:color={colors.lineColor} />
			<ColorControl label="Fill Color" bind:color={colors.fillColor} />
			<ColorControl label="Point Foreground" bind:color={colors.pointFGColor} />
			<ColorControl label="Point Background" bind:color={colors.pointBGColor} />
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
	.dataset-control,
	.camera-info {
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

	.camera-info h4 {
		margin: 0 0 8px 0;
		font-size: 14px;
		font-weight: 600;
		color: #555;
	}

	.camera-coords {
		background: rgba(0, 0, 0, 0.05);
		padding: 8px;
		border-radius: 4px;
		font-family: 'Courier New', monospace;
		font-size: 12px;
		line-height: 1.4;
	}

	.camera-coords div {
		margin-bottom: 2px;
	}

	.camera-coords div:last-child {
		margin-bottom: 0;
	}

	.control-panel :global(.color-control) {
		margin-bottom: 12px;
	}

	.control-panel :global(.color-control:last-child) {
		margin-bottom: 0;
	}
</style>
