<script lang="ts">
	import { onDestroy } from 'svelte';
	import mapray from '@mapray/mapray-js';

	let { url, scene, options = {} }: { url: string; scene: mapray.Scene; options?: any } = $props();

	let loader: mapray.GeoJSONLoader | null = null;

	// optionsやurlが変更されるたびにこの$effectが再実行される
	$effect(() => {
		// optionsが関数の場合、依存関係を明示的に追跡するために呼び出す
		const optionsValue = typeof options === 'function' ? options() : options;

		console.log('Effect triggered, optionsValue:', optionsValue);
		console.log('optionsValue.getFillColor:', optionsValue.getFillColor);

		// optionsが空の場合は処理をスキップ
		if (!optionsValue || !optionsValue.getFillColor) {
			console.log('Skipping: options or getFillColor is not available');
			return;
		}

		// 前回のloaderインスタンスが残っていれば破棄する
		if (loader) {
			loader = null;
		}

		// 新しいオプションでGeoJSONLoaderを再生成する非同期関数
		async function addGeoJSONLayer() {
			try {
				console.log('Creating new GeoJSONLoader with updated options');
				loader = new mapray.GeoJSONLoader(scene, url, {
					// 各コールバック関数は常に最新のoptionsを参照するように修正
					getFillColor: (geojson: any) => {
						const opts = typeof options === 'function' ? options() : options;
						return opts.getFillColor?.(geojson) || geojson.properties?.color || [1.0, 0.0, 0.0];
					},
					getLineColor: (geojson: any) => {
						const opts = typeof options === 'function' ? options() : options;
						return opts.getLineColor?.(geojson) || geojson.properties?.lineColor || [0.0, 0.0, 1.0];
					},
					getPointFGColor: (geojson: any) => {
						const opts = typeof options === 'function' ? options() : options;
						return (
							opts.getPointFGColor?.(geojson) || geojson.properties?.pointFGColor || [0.0, 1.0, 0.0]
						);
					},
					getPointBGColor: (geojson: any) => {
						const opts = typeof options === 'function' ? options() : options;
						return (
							opts.getPointBGColor?.(geojson) || geojson.properties?.pointBGColor || [1.0, 1.0, 0.0]
						);
					},
					getLineWidth: (geojson: any) => {
						const opts = typeof options === 'function' ? options() : options;
						return opts.lineWidth || geojson.properties?.lineWidth || 2;
					},
					getAltitudeMode: (() => {
						const opts = typeof options === 'function' ? options() : options;
						return opts.getAltitudeMode || (() => mapray.AltitudeMode.CLAMP);
					})()
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
