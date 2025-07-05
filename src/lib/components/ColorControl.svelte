<script lang="ts">
	let {
		label = 'Color',
		color = $bindable([0.0, 0.0, 1.0])
	}: { label?: string; color?: number[] } = $props();

	function rgbToHex(rgb: number[]): string {
		const r = Math.round(rgb[0] * 255)
			.toString(16)
			.padStart(2, '0');
		const g = Math.round(rgb[1] * 255)
			.toString(16)
			.padStart(2, '0');
		const b = Math.round(rgb[2] * 255)
			.toString(16)
			.padStart(2, '0');
		return `#${r}${g}${b}`;
	}

	function hexToRgb(hex: string): number[] {
		const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex);
		if (!result) return [0, 0, 0];
		return [
			parseInt(result[1], 16) / 255,
			parseInt(result[2], 16) / 255,
			parseInt(result[3], 16) / 255
		];
	}

	function handleColorChange(event: Event) {
		const target = event.target as HTMLInputElement;
		color = hexToRgb(target.value);
	}

	function handlePresetColor(presetColor: number[]) {
		color = presetColor;
	}

	let hexColor = $derived(rgbToHex(color));
</script>

<div class="color-control">
	<label class="control-label" for="color-picker">{label}</label>
	<div class="color-input-group">
		<input
			id="color-picker"
			type="color"
			bind:value={hexColor}
			oninput={handleColorChange}
			class="color-picker"
		/>
		<div class="color-preview" style="background-color: {hexColor}"></div>
		<span class="color-values">RGB: {color.map((c) => Math.round(c * 255)).join(', ')}</span>
	</div>
	<div class="preset-colors">
		<button
			type="button"
			class="preset-color"
			style="background-color: #0000ff"
			onclick={() => handlePresetColor([0.0, 0.0, 1.0])}
			aria-label="Blue"
		></button>
		<button
			type="button"
			class="preset-color"
			style="background-color: #ff0000"
			onclick={() => handlePresetColor([1.0, 0.0, 0.0])}
			aria-label="Red"
		></button>
		<button
			type="button"
			class="preset-color"
			style="background-color: #00ff00"
			onclick={() => handlePresetColor([0.0, 1.0, 0.0])}
			aria-label="Green"
		></button>
		<button
			type="button"
			class="preset-color"
			style="background-color: #ffff00"
			onclick={() => handlePresetColor([1.0, 1.0, 0.0])}
			aria-label="Yellow"
		></button>
		<button
			type="button"
			class="preset-color"
			style="background-color: #ff00ff"
			onclick={() => handlePresetColor([1.0, 0.0, 1.0])}
			aria-label="Magenta"
		></button>
		<button
			type="button"
			class="preset-color"
			style="background-color: #00ffff"
			onclick={() => handlePresetColor([0.0, 1.0, 1.0])}
			aria-label="Cyan"
		></button>
	</div>
</div>

<style>
	.color-control {
		display: flex;
		flex-direction: column;
		gap: 8px;
		padding: 12px;
		border: 1px solid #ddd;
		border-radius: 6px;
		background: white;
	}

	.control-label {
		font-weight: 600;
		font-size: 14px;
		color: #333;
	}

	.color-input-group {
		display: flex;
		align-items: center;
		gap: 8px;
	}

	.color-picker {
		width: 40px;
		height: 30px;
		border: none;
		border-radius: 4px;
		cursor: pointer;
	}

	.color-preview {
		width: 30px;
		height: 30px;
		border: 1px solid #ccc;
		border-radius: 4px;
	}

	.color-values {
		font-size: 12px;
		color: #666;
		font-family: monospace;
	}

	.preset-colors {
		display: flex;
		gap: 4px;
		flex-wrap: wrap;
	}

	.preset-color {
		width: 24px;
		height: 24px;
		border: 1px solid #ccc;
		border-radius: 4px;
		cursor: pointer;
		transition: transform 0.1s;
	}

	.preset-color:hover {
		transform: scale(1.1);
		border-color: #999;
	}

	.preset-color:active {
		transform: scale(0.95);
	}
</style>
