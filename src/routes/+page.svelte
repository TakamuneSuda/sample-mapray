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

	// updateCameraメソッドをパッチして、rollをサポートする
	console.log('Patching updateCamera method');
	console.log('stdViewer:', stdViewer);
	console.log('updateCamera before patch:', (stdViewer as any).updateCamera);

	const viewer = stdViewer as any;

	viewer.updateCamera = function () {
		console.log('updateCamera called with roll/pitch/yaw:', cameraRollPitchYaw);

		const camera = this.viewer.camera;
		const GeoMath = mapray.GeoMath;

		const mat1 = this._buf_matrix1;
		const mat2 = this._buf_matrix2;
		const mat3 = GeoMath.createMatrix();

		const camera_geoPoint = new mapray.GeoPoint(
			this._camera_parameter.longitude,
			this._camera_parameter.latitude,
			this._camera_parameter.height
		);

		// Roll, Pitch, Yawの回転マトリックスを作成（度数で渡す）
		const pitch_matrix = GeoMath.rotation_matrix([1, 0, 0], cameraRollPitchYaw.pitch, mat1);
		const yaw_matrix = GeoMath.rotation_matrix([0, 0, 1], cameraRollPitchYaw.yaw, mat2);
		const roll_matrix = GeoMath.rotation_matrix([0, 0, 1], cameraRollPitchYaw.roll, mat3);

		// 回転を組み合わせる: Yaw -> Pitch -> Roll
		const temp_matrix = GeoMath.mul_AA(yaw_matrix, pitch_matrix, mat2);
		const eye_matrix = GeoMath.mul_AA(temp_matrix, roll_matrix, mat2);

		// カメラマトリックスを計算して設定
		const camera_matrix = camera_geoPoint.getMlocsToGocsMatrix(mat1);
		GeoMath.mul_AA(camera_matrix, eye_matrix, camera.view_to_gocs);

		// その他のカメラパラメータを設定
		camera.fov = this._camera_parameter.fov;
		camera.near = this._camera_parameter.near;
		camera.far = this._camera_parameter.far;

		// URL書き換え
		this._updateURLHash();
	};

	console.log('updateCamera method patched successfully');
	console.log('updateCamera after patch:', viewer.updateCamera);

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
