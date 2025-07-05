<script lang="ts">
	import { onDestroy } from 'svelte';
	import mapray from '@mapray/mapray-js';

	let {
		url,
		scene,
		options = $bindable({})
	}: { url: string; scene: mapray.Scene; options?: any } = $props();

	let loader: mapray.GeoJSONLoader | null = null;

	// optionsやurlが変更されるたびにこの$effectが再実行される
	$effect(() => {

		console.log(options.getFillColor);
		// 前回のloaderインスタンスが残っていれば破棄する
		if (loader) {
			loader = null;
		}

		// 新しいオプションでGeoJSONLoaderを再生成する非同期関数
		async function addGeoJSONLayer() {
			try {
				loader = new mapray.GeoJSONLoader(scene, url, {
					// 各コールバック関数は常に最新のoptionsを参照するように修正
					getFillColor: (geojson: any) => {
						return options.getFillColor?.(geojson) || geojson.properties?.color || [1.0, 0.0, 0.0];
					},
					getLineColor: (geojson: any) => {
						return (
							options.getLineColor?.(geojson) || geojson.properties?.lineColor || [0.0, 0.0, 1.0]
						);
					},
					getPointFGColor: (geojson: any) => {
						return (
							options.getPointFGColor?.(geojson) ||
							geojson.properties?.pointFGColor || [0.0, 1.0, 0.0]
						);
					},
					getPointBGColor: (geojson: any) => {
						return (
							options.getPointBGColor?.(geojson) ||
							geojson.properties?.pointBGColor || [1.0, 1.0, 0.0]
						);
					},
					getLineWidth: (geojson: any) => {
						return options.lineWidth || geojson.properties?.lineWidth || 2;
					},
					getAltitudeMode: options.getAltitudeMode || (() => mapray.AltitudeMode.CLAMP)
				});
				await loader.load();
			} catch (err) {
				console.error('Failed to load GeoJSON:', err);
			}
		}

		addGeoJSONLayer();

		// $effectのクリーンアップ関数
		return () => {
			if (loader) {
				loader = null;
			}
		};
	});

	// 元のonDestroyは不要になりますが、もし他に破棄処理があれば残します
	onDestroy(() => {
		// $effectのクリーンアップでloaderの破棄は行われるため、
		// ここでは通常不要。
	});
</script>

<!-- {#if isLoading}
	<div class="geojson-loader">Loading GeoJSON...</div>
{/if} -->

<!-- {#if error}
	<div class="geojson-error">Error: {error}</div>
{/if} -->

<style>
	.geojson-loader {
		position: absolute;
		top: 10px;
		right: 10px;
		background: rgba(0, 0, 0, 0.7);
		color: white;
		padding: 8px 12px;
		border-radius: 4px;
		font-size: 12px;
		z-index: 1000;
	}

	.geojson-error {
		position: absolute;
		top: 10px;
		right: 10px;
		background: rgba(220, 53, 69, 0.9);
		color: white;
		padding: 8px 12px;
		border-radius: 4px;
		font-size: 12px;
		z-index: 1000;
	}
</style>
