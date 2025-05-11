<script module lang="ts">
	import { onMount } from 'svelte';
	const textBoxClasses =
		'rounded border border-gray-300 bg-white px-3 py-1 text-base text-gray-700 transition-colors duration-200 ease-in-out outline-none focus:border-indigo-500 focus:ring-2 focus:ring-indigo-200';
	const buttonClasses =
		'rounded border-0 bg-indigo-500 px-6 py-1 text-white hover:bg-indigo-600 focus:outline-none';
	const labelClasses = 'text-sm leading-7 text-gray-600';
	const tooltipClass =
		'max-w-screen pointer-events-none absolute bottom-full mb-2 w-max rounded bg-gray-800 px-2 py-1 text-sm text-white opacity-0 transition-opacity duration-200 group-hover:opacity-100';
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

	let fileName = $state('');
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

	/** チャンネルのマッピング前の値とマッピング後の値 */
	type CurveChannel = {
		before: {
			x: number;
			y: number;
		};
		after: {
			x: number;
			y: number;
		};
	};

	/** glfx用フィルタープロパティ : https://evanw.github.io/glfx.js/docs/ */
	type FilterProps = {
		/** 明るさ */
		brightness: number;
		/** コントラスト */
		contrast: number;
		/** 画像内の色を任意の関数で変換する */
		curves: {
			red: CurveChannel;
			green: CurveChannel;
			blue: CurveChannel;
		};
		/** 色相 */
		hue: number;
		/** 彩度 */
		saturation: number;
		/** セピア */
		sepia: number;
		/** 高周波成分を増幅する画像シャープニング */
		unsharpMask: {
			/** 隣接するピクセルの平均を計算するぼかし半径 */
			radius: number;
			/** スケール係数。0 の場合は効果がなく、値が大きいほど効果が強く */
			strength: number;
		};
		/** 彩度が低い色の彩度を変更(-1 は最小の鮮やかさ、0 は変化なし、1 は最大の鮮やかさ) */
		vibrance: number;
		/** レンズエッジ暗化効果 */
		vignette: {
			/** フレームの中央は0、フレームの端は1 */
			size: number;
			/** 0は効果なし、1はレンズを最大限暗くする */
			amount: number;
		};
		/** 特定の閾値よりも強いエッジを暗くする */
		ink: number;
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
		brightness: 0.1,
		contrast: 0.3,
		curves: {
			red: {
				before: {
					x: 0,
					y: 0
				},
				after: {
					x: 1,
					y: 1
				}
			},
			green: {
				before: {
					x: 0,
					y: 0
				},
				after: {
					x: 1,
					y: 1
				}
			},
			blue: {
				before: {
					x: 0,
					y: 0
				},
				after: {
					x: 1,
					y: 1
				}
			}
		},
		hue: 0.02,
		saturation: 0.2,
		sepia: 0,
		unsharpMask: {
			radius: 12,
			strength: 0.4
		},
		vibrance: -0.1,
		vignette: {
			size: 0,
			amount: 0.5
		},
		ink: 0,
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
		fileName = file?.name.replace(/\.[^/.]+$/, '') ?? '';
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
			.curves(
				[
					[filterProps.curves.red.before.x, filterProps.curves.red.before.y],
					[filterProps.curves.red.after.x, filterProps.curves.red.after.y]
				],
				[
					[filterProps.curves.green.before.x, filterProps.curves.green.before.y],
					[filterProps.curves.green.after.x, filterProps.curves.green.after.y]
				],
				[
					[filterProps.curves.blue.before.x, filterProps.curves.blue.before.y],
					[filterProps.curves.blue.after.x, filterProps.curves.blue.after.y]
				]
			)
			.vignette(filterProps.vignette.size, filterProps.vignette.amount)
			.sepia(filterProps.sepia)
			.vibrance(filterProps.vibrance)
			.unsharpMask(filterProps.unsharpMask.radius, filterProps.unsharpMask.strength)
			.ink(filterProps.ink)
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
		a.download = `${fileName} - cine-sco.png`;
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
	const size = 100;
	const toX = (x: number) => x * size;
	const toY = (y: number) => (1 - y) * size; // SVGのy軸反転
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
			<div class="group relative inline-block">
				<label for="brightness" class={`${labelClasses}`}>明るさ</label>
				<div class={`${tooltipClass}`}>明るさを加算します。</div>
			</div>
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
			<div class="group relative inline-block">
				<label for="contrast" class={`${labelClasses}`}>コントラスト</label>
				<div class={`${tooltipClass}`}>コントラストを乗算します。</div>
			</div>
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
			<div class="group relative col-span-full inline-block">
				<div class={`${labelClasses}`}>カーブ(赤・緑・青チャンネル)</div>
				<div class={`${tooltipClass}`}>画像の色を２次元の点の集合の間で補間します。</div>
			</div>
			<div class="col-span-full grid grid-cols-3 gap-4">
				<div style="width: fit-content;">
					<!-- 上スライダー（after.x） -->
					<div style="text-align: center; margin-right: 5px;">
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							bind:value={filterProps.curves.red.after.x}
						/>
					</div>

					<div style="display: flex; align-items: center;">
						<!-- 左スライダー（before.y） -->
						<div style="writing-mode: vertical-lr; transform: rotate(180deg); margin-right: 5px;">
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={filterProps.curves.red.before.y}
							/>
						</div>

						<!-- SVG描画エリア -->
						<svg width={size} height={size} style="border: 1px solid #ccc; background: #f8f8f8;">
							<!-- 点を結ぶ線 -->
							<line
								x1={toX(filterProps.curves.red.before.x)}
								y1={toY(filterProps.curves.red.before.y)}
								x2={toX(filterProps.curves.red.after.x)}
								y2={toY(filterProps.curves.red.after.y)}
								stroke="red"
								stroke-width="2"
							/>

							<!-- before点 -->
							<circle
								cx={toX(filterProps.curves.red.before.x)}
								cy={toY(filterProps.curves.red.before.y)}
								r="5"
								fill="red"
							/>

							<!-- after点 -->
							<circle
								cx={toX(filterProps.curves.red.after.x)}
								cy={toY(filterProps.curves.red.after.y)}
								r="5"
								fill="red"
							/>
						</svg>

						<!-- 右スライダー（after.y） -->
						<div style="writing-mode: sideways-lr; margin-left: 5px;">
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={filterProps.curves.red.after.y}
							/>
						</div>
					</div>

					<!-- 下スライダー（before.x） -->
					<div style="text-align: center; margin-top: 5px;">
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							bind:value={filterProps.curves.red.before.x}
						/>
					</div>
				</div>
				<div style="width: fit-content;">
					<!-- 上スライダー（after.x） -->
					<div style="text-align: center; margin-right: 5px;">
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							bind:value={filterProps.curves.green.after.x}
						/>
					</div>

					<div style="display: flex; align-items: center;">
						<!-- 左スライダー（before.y） -->
						<div style="writing-mode: vertical-lr; transform: rotate(180deg); margin-right: 5px;">
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={filterProps.curves.green.before.y}
							/>
						</div>

						<!-- SVG描画エリア -->
						<svg width={size} height={size} style="border: 1px solid #ccc; background: #f8f8f8;">
							<!-- 点を結ぶ線 -->
							<line
								x1={toX(filterProps.curves.green.before.x)}
								y1={toY(filterProps.curves.green.before.y)}
								x2={toX(filterProps.curves.green.after.x)}
								y2={toY(filterProps.curves.green.after.y)}
								stroke="green"
								stroke-width="2"
							/>

							<!-- before点 -->
							<circle
								cx={toX(filterProps.curves.green.before.x)}
								cy={toY(filterProps.curves.green.before.y)}
								r="5"
								fill="green"
							/>

							<!-- after点 -->
							<circle
								cx={toX(filterProps.curves.green.after.x)}
								cy={toY(filterProps.curves.green.after.y)}
								r="5"
								fill="green"
							/>
						</svg>

						<!-- 右スライダー（after.y） -->
						<div style="writing-mode: sideways-lr; margin-left: 5px;">
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={filterProps.curves.green.after.y}
							/>
						</div>
					</div>

					<!-- 下スライダー（before.x） -->
					<div style="text-align: center; margin-top: 5px;">
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							bind:value={filterProps.curves.green.before.x}
						/>
					</div>
				</div>
				<div style="width: fit-content;">
					<!-- 上スライダー（after.x） -->
					<div style="text-align: center; margin-right: 5px;">
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							bind:value={filterProps.curves.blue.after.x}
						/>
					</div>

					<div style="display: flex; align-items: center;">
						<!-- 左スライダー（before.y） -->
						<div style="writing-mode: vertical-lr; transform: rotate(180deg); margin-right: 5px;">
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={filterProps.curves.blue.before.y}
							/>
						</div>

						<!-- SVG描画エリア -->
						<svg width={size} height={size} style="border: 1px solid #ccc; background: #f8f8f8;">
							<!-- 点を結ぶ線 -->
							<line
								x1={toX(filterProps.curves.blue.before.x)}
								y1={toY(filterProps.curves.blue.before.y)}
								x2={toX(filterProps.curves.blue.after.x)}
								y2={toY(filterProps.curves.blue.after.y)}
								stroke="blue"
								stroke-width="2"
							/>

							<!-- before点 -->
							<circle
								cx={toX(filterProps.curves.blue.before.x)}
								cy={toY(filterProps.curves.blue.before.y)}
								r="5"
								fill="blue"
							/>

							<!-- after点 -->
							<circle
								cx={toX(filterProps.curves.blue.after.x)}
								cy={toY(filterProps.curves.blue.after.y)}
								r="5"
								fill="blue"
							/>
						</svg>

						<!-- 右スライダー（after.y） -->
						<div style="writing-mode: sideways-lr; margin-left: 5px;">
							<input
								type="range"
								min="0"
								max="1"
								step="0.01"
								bind:value={filterProps.curves.blue.after.y}
							/>
						</div>
					</div>

					<!-- 下スライダー（before.x） -->
					<div style="text-align: center; margin-top: 5px;">
						<input
							type="range"
							min="0"
							max="1"
							step="0.01"
							bind:value={filterProps.curves.green.before.x}
						/>
					</div>
				</div>
			</div>
			<div class="group relative inline-block">
				<label for="sepia" class={`${labelClasses}`}>セピア</label>
				<div class={`${tooltipClass}`}>
					古い写真を模した赤茶色のモノクローム色調を画像に与えます。
				</div>
			</div>
			<input
				type="number"
				id="sepia"
				name="sepia"
				min="0"
				max="1"
				step="0.01"
				bind:value={filterProps.sepia}
				class={`w-full ${textBoxClasses}`}
			/>
			<div class="group relative inline-block">
				<label for="hue" class={`${labelClasses}`}>色相</label>
				<div class={`${tooltipClass}`}>
					黒（0、0、0）から白（1、1、1）までの直線であるグレースケール線を中心に色ベクトルを回転させます。
				</div>
			</div>
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
			<div class="group relative inline-block">
				<label for="saturation" class={`${labelClasses}`}>彩度</label>
				<div class={`${tooltipClass}`}>
					すべてのカラーチャンネル値を平均カラーチャンネル値に向かって、または平均カラーチャンネル値から離れるようにスケーリングします。
				</div>
			</div>
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
			<div class="group relative inline-block">
				<label for="vibrance" class={`${labelClasses}`}>自然な彩度</label>
				<div class={`${tooltipClass}`}>
					彩度の低い色の彩度を変更します、彩度の高い色は変更しません。<br />
					-1 は最小の鮮やかさ、0 は変化なし、1 は最大の鮮やかさ
				</div>
			</div>
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
			<div class="group relative inline-block">
				<label for="ink" class={`${labelClasses}`}>エッジ</label>
				<div class={`${tooltipClass}`}>
					ある閾値より強いエッジを暗くすることで、画像の輪郭をインクでシミュレートします。
					<br
					/>エッジ検出値は、それぞれ異なる半径のぼかしを使ってぼかした画像の2つのコピーの差です。
					<br
					/>通常は0から1の範囲の値で十分で、0では画像は変化せず、1では黒いエッジがたくさん追加されます。<br
					/>負の値を指定すると、インクのエッジは黒ではなく白になります。
				</div>
			</div>
			<input
				type="number"
				id="ink"
				name="ink"
				min="-1"
				max="1"
				step="0.01"
				bind:value={filterProps.ink}
				class={`w-full ${textBoxClasses}`}
			/>
			<div class="group relative inline-block"></div>
			<div></div>
			<div class="group relative inline-block">
				<label for="unsharpMask.radius" class={`${labelClasses}`}>明瞭度：半径</label>
				<div class={`${tooltipClass}`}>
					画像の高周波を増幅するシャープニングの一種。ピクセルを隣接するピクセルの平均値からスケーリングします。<br
					/>
					半径：隣接ピクセルの平均を計算するぼかし半径。
				</div>
			</div>
			<input
				type="number"
				id="unsharpMask.radius"
				name="unsharpMask.radius"
				min="0"
				step="1"
				bind:value={filterProps.unsharpMask.radius}
				class={`w-full ${textBoxClasses}`}
			/>
			<div class="group relative inline-block">
				<label for="unsharpMask.strength" class={`${labelClasses}`}>明瞭度：強度</label>
				<div class={`${tooltipClass}`}>
					画像の高周波を増幅するシャープニングの一種。ピクセルを隣接するピクセルの平均値からスケーリングします。<br
					/>
					強度：0は効果がなく、値が大きいほど効果が強くなる。
				</div>
			</div>
			<input
				type="number"
				id="unsharpMask.strength"
				name="unsharpMask.strength"
				min="0"
				step="0.1"
				bind:value={filterProps.unsharpMask.strength}
				class={`w-full ${textBoxClasses}`}
			/>
			<div class="group relative inline-block">
				<label for="vignette.size" class={`${labelClasses}`}>ビネット加工：位置</label>
				<div class={`${tooltipClass}`}>
					画像の端を暗くする効果をシミュレートします。
					<br />位置：フレームの中央は0、フレームの端は1
				</div>
			</div>
			<input
				type="number"
				id="vignette.size"
				name="vignette.size"
				min="0"
				max="1"
				step="0.01"
				bind:value={filterProps.vignette.size}
				class={`w-full ${textBoxClasses}`}
			/>
			<div class="group relative inline-block">
				<label for="vignette.amount" class={`${labelClasses}`}>ビネット加工：強度</label>
				<div class={`${tooltipClass}`}>
					画像の端を暗くする効果をシミュレートします。
					<br />強度：0は効果なし、1はレンズを最大に暗くする
				</div>
			</div>
			<input
				type="number"
				id="vignette.amount"
				name="vignette.amount"
				min="0"
				max="1"
				step="0.01"
				bind:value={filterProps.vignette.amount}
				class={`w-full ${textBoxClasses}`}
			/>
			<div class="group relative inline-block">
				<label for="bulgePinch.radius" class={`${labelClasses}`}>レンズ膨らみ効果：半径</label>
				<div class={`${tooltipClass}`}>
					画像を円形に膨らませたり、つまんだりする。
					<br />半径：効果円の半径
				</div>
			</div>
			<input
				type="number"
				id="bulgePinch.radius"
				name="bulgePinch.radius"
				min="0"
				step="0.1"
				bind:value={filterProps.bulgePinch.radius}
				class={`w-full ${textBoxClasses}`}
			/>
			<div class="group relative inline-block">
				<label for="bulgePinch.strength" class={`${labelClasses}`}>レンズ膨らみ効果：強度</label>
				<div class={`${tooltipClass}`}>
					画像を円形に膨らませたり、つまんだりする。
					<br />強度：-1 は強いつまみ、0 は効果なし、1 は強い膨らみ
				</div>
			</div>
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
