<script module lang="ts">
	import { onMount } from 'svelte';
	const textBoxClasses =
		'rounded border border-gray-300 bg-white px-3 py-1 text-base text-gray-700 transition-colors duration-200 ease-in-out outline-none focus:border-indigo-500 focus:ring-2 focus:ring-indigo-200';
	const buttonClasses =
		'rounded border-0 bg-indigo-500 px-6 py-1 text-white hover:bg-indigo-600 focus:outline-none';
	const labelClasses = 'text-sm leading-7 text-gray-600';
</script>

<script lang="ts">
	import {
		Canvas,
		FabricImage,
		FabricText,
		type ImageProps,
		type ObjectEvents,
		type SerializedImageProps
	} from 'fabric';

	/** 表示用縮尺倍率 */
	let multiplier = $state(5);
	/** 原寸幅 */
	let originalWidth = $state(0);
	/** 原寸高さ */
	let originalHeight = $state(0);
	/** 画面表示用幅 */
	let previewWidth = $derived(originalWidth / multiplier);
	/** 画面表示用高さ */
	let previewHeight = $derived(originalHeight / multiplier);
	/** シネスコ黒帯高さ */
	let cinemaPadding = $state(0);
	let canvasElement: HTMLCanvasElement;
	let canvas: Canvas;
	let subtitleTop = $state('');
	let subtitleBottom = $state('');
	let subtitleRight = $state('');

	let processing = $state(false);

	/** 結果を保存する画像バッファ(glfx用) */
	let fxCanvas: any;
	/** 画像データの生のソース(glfx用) */
	let texture: any;

	/** glfx用フィルタープロパティ : https://evanw.github.io/glfx.js/docs/ */
	type FilterProps = {
		/** 明るさ */
		brightness: number;
		/** コントラスト */
		contrast: number;
		/** 色相 */
		hue: number;
		/** 彩度 */
		saturation: number;
		/** セピア */
		sepia: number;
		/** 彩度が低い色の彩度を変更(-1 は最小の鮮やかさ、0 は変化なし、1 は最大の鮮やかさ) */
		vibrance: number;
		/** レンズエッジ暗化効果 */
		vignette: {
			/** フレームの中央は0、フレームの端は1 */
			size: number;
			/** 0は効果なし、1はレンズを最大限暗くする */
			amount: number;
		};
		/** 画像を円形に膨らませたり、つまんだり */
		bulgePinch: {
			/** 効果の円の半径 */
			radius: number;
			/** -1 は強い挟み込み、0 は効果なし、1 は強い膨らみ */
			strength: number;
		};
	};

	/** フィルターオプション(glfx用) */
	let filterProps: FilterProps = $state({
		brightness: 0,
		contrast: 0,
		hue: 0.05,
		saturation: 0.2,
		sepia: 0,
		vibrance: 0,
		vignette: {
			size: 0,
			amount: 0
		},
		bulgePinch: {
			radius: 200,
			strength: 0
		}
	} as FilterProps);

	onMount(async () => {
		// 初期化(だいたいこんなもんというサイズで)
		canvasElement.width = 768;
		canvasElement.height = 432;
		canvas = new Canvas(canvasElement);

		document.addEventListener('keydown', (event) => {
			handleKeyDown(event);
		});
	});

	function addSubtitleTop(): void {
		const textObj = new FabricText(subtitleTop, {
			left: previewWidth / 2,
			top: cinemaPadding,
			fontSize: 28,
			fill: 'white',
			textAlign: 'center',
			originX: 'center',
			fontFamily: 'M PLUS Rounded 1c'
		});

		canvas.add(textObj);
		canvas.renderAll();
	}

	function addSubtitleBottom(): void {
		const textObj = new FabricText(subtitleBottom, {
			left: previewWidth / 2,
			top: (originalHeight - cinemaPadding) / multiplier - 28 - 6,
			fontSize: 28,
			fill: 'white',
			textAlign: 'center',
			originX: 'center',
			fontFamily: 'M PLUS Rounded 1c'
		});

		canvas.add(textObj);
		canvas.renderAll();
	}

	function addSubtitleRight(): void {
		const baseLeft = previewWidth - 60;
		const baseTop = cinemaPadding / multiplier + 24;
		const fontSize = 24;
		const lineHeight = 24;

		// 1文字ずつ縦に並べる
		for (let i = 0; i < subtitleRight.length; i++) {
			const char = subtitleRight[i];
			const textObj = new FabricText(char, {
				left: baseLeft,
				top: baseTop + i * lineHeight,
				fontSize,
				// fontStyle: 'italic', // なんか日本語だと変
				fill: 'white',
				textAlign: 'center',
				originX: 'center'
			});
			canvas.add(textObj);
		}
		canvas.renderAll();
	}

	function handleImageUpload(event: Event): void {
		processing = true;
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		const reader = new FileReader();
		reader.onload = () => {
			const img = new Image();
			img.onload = async () => {
				originalWidth = img.width;
				originalHeight = img.height;

				// 画面に収まるような倍率設定
				multiplier = Math.min(originalWidth / 768, originalHeight / 432);
				// 21:9の縦サイズを計算（元画像の横幅に対して）
				const targetHeight = Math.floor((originalWidth * 9) / 21);
				// 元画像のアスペクト比が16:9（縦が大きい）と仮定して上下に黒帯を追加
				cinemaPadding = Math.floor((originalHeight - targetHeight) / 2);
				const fabricImg = await cinesco(img);

				const cinescoImageUrl = fabricImg.toDataURL();
				const imgWithBlackBars = new Image();
				imgWithBlackBars.onload = () => {
					fxCanvas = (window as any).fx.canvas();
					texture = fxCanvas.texture(imgWithBlackBars);
					fxCanvas.draw(texture).update();
				};
				imgWithBlackBars.src = cinescoImageUrl;

				// スケーリング処理
				scalingImage(fabricImg);

				// Fabricキャンバスを再作成
				canvasElement.width = previewWidth;
				canvasElement.height = previewHeight;
				canvas.dispose();
				canvas = new Canvas(canvasElement);
				canvas.add(fabricImg);
				canvas.sendObjectToBack(fabricImg);
				canvas.renderAll();
				processing = false;
			};
			img.src = reader.result as string;
		};
		reader.readAsDataURL(file);
		processing = false;
	}

	async function cinesco(img: HTMLImageElement) {
		// 黒帯つきキャンバス作成
		const offscreen = document.createElement('canvas');
		offscreen.width = originalWidth;
		offscreen.height = originalHeight;

		const ctx = offscreen.getContext('2d');
		if (ctx) {
			// 元画像を貼り付け
			ctx.drawImage(img, 0, 0);
			// 黒帯作成
			ctx.fillStyle = 'black';
			ctx.fillRect(0, 0, originalWidth, cinemaPadding);
			ctx.fillRect(0, originalHeight - cinemaPadding, originalWidth, cinemaPadding);
		}
		// DataURLをFabricImageとして読み込み
		const dataUrl = offscreen.toDataURL();
		const fabricImg = await FabricImage.fromURL(dataUrl, {}, { selectable: false });
		return fabricImg;
	}

	function scalingImage(img: FabricImage<Partial<ImageProps>, SerializedImageProps, ObjectEvents>) {
		img.scaleToWidth(previewWidth);
		img.scaleToHeight(previewHeight);
		img.set({ left: 0, top: 0 });
	}

	function applyFilter() {
		processing = true;
		if (!fxCanvas || !texture) return;

		// 実際には以下を調整してティール＆オレンジ風に
		fxCanvas
			.draw(texture)
			.brightnessContrast(filterProps.brightness, filterProps.contrast)
			.bulgePinch(
				fxCanvas.width / 2,
				fxCanvas.height / 2,
				filterProps.bulgePinch.radius,
				filterProps.bulgePinch.strength
			)
			.hueSaturation(filterProps.hue, filterProps.saturation)
			.vignette(filterProps.vignette.size, filterProps.vignette.amount)
			.sepia(filterProps.sepia)
			.vibrance(filterProps.vibrance)
			.update();

		const resultURL = fxCanvas.toDataURL();
		replaceImageOnFabric(resultURL);
	}

	function replaceImageOnFabric(dataURL: string) {
		FabricImage.fromURL(dataURL).then((newImg) => {
			const active = canvas.getActiveObject();
			if (active && active.type === 'image') {
				canvas.remove(active);
			}
			// スケーリング処理
			newImg.selectable = false;
			scalingImage(newImg);
			deleteImage();
			canvas.add(newImg);
			canvas.sendObjectToBack(newImg);
			canvas.renderAll();
			processing = false;
		});
	}

	function downloadImage(): void {
		// 原寸Download
		const imageSrc = canvas.toDataURL({ multiplier });

		const a = document.createElement('a');
		a.href = imageSrc;
		a.download = 'image.png';
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
	}

	function handleKeyDown(event: KeyboardEvent) {
		if (event.key === 'Delete' || event.key === 'Backspace') {
			const activeObjects = canvas.getActiveObjects();
			activeObjects.map((obj) => {
				canvas.remove(obj);
				canvas.renderAll();
			});
		}
	}

	function deleteImage() {
		const objects = canvas.getObjects();
		objects.map((obj) => {
			if (obj.type === 'image') {
				canvas.remove(obj);
				canvas.renderAll();
				return;
			}
		});
	}
</script>

<svelte:head>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" />
	<link
		href="https://fonts.googleapis.com/css2?family=M+PLUS+Rounded+1c&display=swap"
		rel="stylesheet"
	/>
	<script src="/glfx.js"></script>
</svelte:head>
{#if processing}
	<div class="loading-overlay">
		<div class="loading">
			<span></span>
			<span></span>
			<span></span>
			<span></span>
		</div>
	</div>
{/if}
<section class="container max-w-[49rem] p-2">
	<div class="grid grid-cols-1 gap-2 md:grid-cols-[3rem_auto_10rem] md:gap-4">
		<div class="col-span-full">
			<h1 class="title-font mb-1 text-lg font-medium text-gray-900">シネマ風字幕挿入サービス</h1>
			<p class="leading-relaxed text-gray-600">
				画像の上下に黒帯を入れたり文字を入れてダウンロードできます!
			</p>
		</div>
		<div class="col-span-full overflow-auto md:justify-items-center">
			<div class="w-full pb-5 md:pb-0">
				<canvas
					bind:this={canvasElement}
					onkeydown={(e) => handleKeyDown(e)}
					tabindex={0}
					class="border-1"
				></canvas>
			</div>
		</div>
		<div class="col-span-full">
			<h3 class={`${labelClasses}`}>字幕テキスト</h3>
		</div>
		<label for="subtitleTop" class={`${labelClasses}`}>上側</label>
		<input
			type="text"
			id="subtitleTop"
			name="subtitleTop"
			bind:value={subtitleTop}
			class={`w-full ${textBoxClasses}`}
		/>
		<button onclick={addSubtitleTop} class={`w-full ${buttonClasses}`}>挿入</button>
		<label for="subtitleBottom" class={`${labelClasses}`}>下側</label>
		<input
			type="text"
			id="subtitleBottom"
			name="subtitleBottom"
			bind:value={subtitleBottom}
			class={`w-full ${textBoxClasses}`}
		/>
		<button onclick={addSubtitleBottom} class={buttonClasses}>挿入</button>
		<label for="subtitleRight" class={`${labelClasses}`}>右側縦</label>
		<input
			type="text"
			id="subtitleRight"
			name="subtitleRight"
			bind:value={subtitleRight}
			class={`w-full ${textBoxClasses}`}
		/>
		<button onclick={addSubtitleRight} class={`w-full ${buttonClasses}`}>挿入</button>
		<label for="gazo" class={`${labelClasses}`}>写真</label>
		<input
			type="file"
			id="gazo"
			name="gazo"
			accept="image/*"
			onchange={handleImageUpload}
			class="w-full cursor-pointer rounded border border-gray-300 px-3 text-sm leading-8 focus:outline-none"
		/>
		<button onclick={deleteImage} class={`w-full ${buttonClasses}`}>画像を削除</button>
		<div class={`${labelClasses}`}>加工</div>
		<div class="grid grid-cols-[10rem_auto] gap-2 md:grid-cols-[10rem_auto_10rem_auto] md:gap-4">
			<label for="brightness" class={`${labelClasses}`}>明るさ</label>
			<input
				type="number"
				id="brightness"
				name="brightness"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.brightness}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="contrast" class={`${labelClasses}`}>コントラスト</label>
			<input
				type="number"
				id="contrast"
				name="contrast"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.contrast}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="hue" class={`${labelClasses}`}>色相</label>
			<input
				type="number"
				id="hue"
				name="hue"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.hue}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="sepia" class={`${labelClasses}`}>セピア</label>
			<input
				type="number"
				id="sepia"
				name="sepia"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.sepia}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="saturation" class={`${labelClasses}`}>彩度</label>
			<input
				type="number"
				id="saturation"
				name="saturation"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.saturation}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="vibrance" class={`${labelClasses}`}>自然な彩度</label>
			<input
				type="number"
				id="vibrance"
				name="vibrance"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.vibrance}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="vignette.size" class={`${labelClasses}`}>ビネット：大きさ</label>
			<input
				type="number"
				id="vignette.size"
				name="vignette.size"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.vignette.size}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="vignette.amount" class={`${labelClasses}`}>ビネット：効果量</label>
			<input
				type="number"
				id="vignette.amount"
				name="vignette.amount"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.vignette.amount}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="bulgePinch.radius" class={`${labelClasses}`}>レンズ膨らみ効果：半径</label>
			<input
				type="number"
				id="bulgePinch.radius"
				name="bulgePinch.radius"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.bulgePinch.radius}
				class={`w-full ${textBoxClasses}`}
			/>
			<label for="bulgePinch.strength" class={`${labelClasses}`}>レンズ膨らみ効果：強度</label>
			<input
				type="number"
				id="bulgePinch.strength"
				name="bulgePinch.strength"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.bulgePinch.strength}
				class={`w-full ${textBoxClasses}`}
			/>
		</div>
		<button onclick={applyFilter} class={buttonClasses}>実行</button>
		<div class="col-span-full">
			<p class="text-xs text-gray-500">
				※画面に挿入したものは選んでデリートキーかバックスペースキーで消せます。
			</p>
		</div>
		<div class="col-span-full md:col-span-2">
			<button onclick={downloadImage} class={`w-full ${buttonClasses}`}>ダウンロード</button>
		</div>
		<button
			onclick={() => {
				window.location.reload();
			}}
			class={`w-full ${buttonClasses}`}>リセット</button
		>
	</div>
</section>

<style>
	* {
		font-family: 'M PLUS Rounded 1c', sans-serif;
		font-weight: 400;
		font-style: normal;
	}

	/* From Uiverse.io by terenceodonoghue */
	.loading-overlay {
		z-index: 9999;
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background-color: rgba(0, 0, 0, 0.5);
	}
	.loading {
		position: absolute;
		top: 50%;
		left: 50%;
		border-radius: 50%;
		height: 96px;
		width: 96px;
		animation: rotate_3922 1.2s linear infinite;
		background-color: #9b59b6;
		background-image: linear-gradient(#9b59b6, #84cdfa, #5ad1cd);
	}

	.loading span {
		position: absolute;
		border-radius: 50%;
		height: 100%;
		width: 100%;
		background-color: #9b59b6;
		background-image: linear-gradient(#9b59b6, #84cdfa, #5ad1cd);
	}

	.loading span:nth-of-type(1) {
		filter: blur(5px);
	}

	.loading span:nth-of-type(2) {
		filter: blur(10px);
	}

	.loading span:nth-of-type(3) {
		filter: blur(25px);
	}

	.loading span:nth-of-type(4) {
		filter: blur(50px);
	}

	.loading::after {
		content: '';
		position: absolute;
		top: 10px;
		left: 10px;
		right: 10px;
		bottom: 10px;
		background-color: #fff;
		border: solid 5px #ffffff;
		border-radius: 50%;
	}

	@keyframes rotate_3922 {
		from {
			transform: translate(-50%, -50%) rotate(0deg);
		}

		to {
			transform: translate(-50%, -50%) rotate(360deg);
		}
	}
</style>
