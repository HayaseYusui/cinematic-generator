<script module lang="ts">
	import { onMount } from 'svelte';
	const textBoxClasses =
		'rounded border border-gray-300 bg-white px-3 py-1 text-base text-gray-700 transition-colors duration-200 ease-in-out outline-none focus:border-indigo-500 focus:ring-2 focus:ring-indigo-200';
	const buttonClasses =
		'rounded border-0 bg-indigo-500 px-6 py-1 text-white hover:bg-indigo-600 focus:outline-none';
	const labelClasses = 'text-sm leading-7 text-gray-600';
</script>

<script lang="ts">
	import { Canvas, FabricImage, FabricText } from 'fabric';
	/** 表示用縮尺倍率 */
	let multiplier = $state(5);
	let initWidth = $state(0);
	let initHeight = $state(0);
	let cinemaPadding = $state(0);
	let prevWidth = $derived(initWidth / multiplier);
	let prevHeight = $derived(initHeight / multiplier);
	let canvasElement: HTMLCanvasElement;
	let canvas: Canvas;
	let subtitleTop: string = $state('');
	let subtitleBottom: string = $state('');
	let subtitleRight: string = $state('');

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
			left: prevWidth / 2,
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
			left: prevWidth / 2,
			top: (initHeight - cinemaPadding) / multiplier - 28 - 6,
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
		const baseLeft = prevWidth - 60;
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
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];
		if (!file) return;

		const reader = new FileReader();
		reader.onload = () => {
			const img = new Image();
			img.onload = async () => {
				initWidth = img.width;
				initHeight = img.height;

				// 画面に収まるような倍率設定
				multiplier = Math.min(initWidth / 768, initHeight / 432);

				console.log(multiplier);
				// 21:9の縦サイズを計算（元画像の横幅に対して）
				const targetHeight = Math.floor((initWidth * 9) / 21);

				// 元画像のアスペクト比が16:9（縦が大きい）と仮定して上下に黒帯を追加
				cinemaPadding = Math.floor((initHeight - targetHeight) / 2);

				// 黒帯つきキャンバス作成
				const offscreen = document.createElement('canvas');
				offscreen.width = initWidth;
				offscreen.height = initHeight;

				const ctx = offscreen.getContext('2d');
				if (!ctx) return;

				// 元画像を貼り付け
				ctx.drawImage(img, 0, 0);
				// 黒帯作成
				ctx.fillStyle = 'black';
				ctx.fillRect(0, 0, initWidth, cinemaPadding);
				ctx.fillRect(0, initHeight - cinemaPadding, initWidth, cinemaPadding);

				// DataURLをFabricImageとして読み込み
				const dataUrl = offscreen.toDataURL();
				const fabricImg = await FabricImage.fromURL(dataUrl, {}, { selectable: false });

				// スケーリング処理
				fabricImg.scaleToWidth(prevWidth);
				fabricImg.scaleToHeight(prevHeight);
				fabricImg.set({ left: 0, top: 0 });
				console.log(prevWidth);
				// Fabricキャンバスを再作成
				canvasElement.width = prevWidth;
				canvasElement.height = prevHeight;
				canvas.dispose();
				canvas = new Canvas(canvasElement);
				canvas.add(fabricImg);
				canvas.renderAll();
			};
			img.src = reader.result as string;
		};
		reader.readAsDataURL(file);
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
</svelte:head>
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
</style>
