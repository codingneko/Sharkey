<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<div :class="$style.root">
	<XBanner v-for="media in mediaList.filter(media => !previewable(media))" :key="media.id" :media="media"/>
	<div v-if="mediaList.filter(media => previewable(media)).length > 0" :class="$style.container">
		<div
			ref="gallery"
			:class="[
				$style.medias,
				count === 1 ? [$style.n1, {
					[$style.n116_9]: prefer.s.mediaListWithOneImageAppearance === '16_9',
					[$style.n11_1]: prefer.s.mediaListWithOneImageAppearance === '1_1',
					[$style.n12_3]: prefer.s.mediaListWithOneImageAppearance === '2_3',
				}] : count === 2 ? $style.n2 : count === 3 ? $style.n3 : count === 4 ? $style.n4 : $style.nMany,
			]"
		>
			<template v-for="media in mediaList.filter(media => previewable(media))">
				<XVideo v-if="media.type.startsWith('video')" :key="`video:${media.id}`" :class="$style.media" :video="media"/>
				<XImage v-else-if="media.type.startsWith('image')" :key="`image:${media.id}`" :class="$style.media" class="image" :data-id="media.id" :image="media" :raw="raw"/>
				<XModPlayer v-else-if="isModule(media)" :key="`module:${media.id}`" :class="$style.media" :module="media"/>
				<XFlashPlayer v-else-if="isFlash(media)" :key="`flash:${media.id}`" :class="$style.media" :flashFile="media"/>
			</template>
		</div>
	</div>
</div>
</template>

<script lang="ts" setup>
import { computed, onMounted, onUnmounted, useTemplateRef } from 'vue';
import * as Misskey from 'misskey-js';
import PhotoSwipeLightbox from 'photoswipe/lightbox';
import PhotoSwipe from 'photoswipe';
import 'photoswipe/style.css';
import '@photo-sphere-viewer/core/index.css';
import { FILE_TYPE_BROWSERSAFE, FILE_EXT_TRACKER_MODULES, FILE_TYPE_TRACKER_MODULES, FILE_TYPE_FLASH_CONTENT, FILE_EXT_FLASH_CONTENT } from '@@/js/const.js';
import XBanner from '@/components/MkMediaBanner.vue';
import XImage from '@/components/MkMediaImage.vue';
import XVideo from '@/components/MkMediaVideo.vue';
import XModPlayer from '@/components/SkModPlayer.vue';
import XFlashPlayer from '@/components/SkFlashPlayer.vue';
import * as os from '@/os.js';
import { focusParent } from '@/utility/focus.js';
import { prefer } from '@/preferences.js';

const props = defineProps<{
	mediaList: Misskey.entities.DriveFile[];
	raw?: boolean;
}>();

const gallery = useTemplateRef('gallery');
const pswpZIndex = os.claimZIndex('middle');
window.document.documentElement.style.setProperty('--mk-pswp-root-z-index', pswpZIndex.toString());
const count = computed(() => props.mediaList.filter(media => previewable(media)).length);
let lightbox: PhotoSwipeLightbox | null = null;

let activeEl: HTMLElement | null = null;

const popstateHandler = (): void => {
	if (lightbox?.pswp && lightbox.pswp.isOpen === true) {
		lightbox.pswp.close();
	}
};

async function calcAspectRatio() {
	if (!gallery.value) return;

	const img = props.mediaList[0];

	if (props.mediaList.length !== 1 || !(img.properties.width && img.properties.height)) {
		gallery.value.style.aspectRatio = '';
		return;
	}

	const ratioMax = (ratio: number) => {
		if (img.properties.width == null || img.properties.height == null) return '';
		return `${Math.max(ratio, img.properties.width / img.properties.height).toString()} / 1`;
	};

	switch (prefer.s.mediaListWithOneImageAppearance) {
		case '16_9':
			gallery.value.style.aspectRatio = ratioMax(16 / 9);
			break;
		case '1_1':
			gallery.value.style.aspectRatio = ratioMax(1 / 1);
			break;
		case '2_3':
			gallery.value.style.aspectRatio = ratioMax(2 / 3);
			break;
		default:
			gallery.value.style.aspectRatio = '';
			break;
	}
}

const isModule = (file: Misskey.entities.DriveFile): boolean => {
	return FILE_TYPE_TRACKER_MODULES.includes(file.type) || FILE_EXT_TRACKER_MODULES.some((ext) => {
		return (file.name.toLowerCase().endsWith('.' + ext) || file.name.toLowerCase().endsWith('.' + ext + '.unknown'));
	});
};

const isFlash = (file: Misskey.entities.DriveFile): boolean => {
	return FILE_TYPE_FLASH_CONTENT.includes(file.type) || FILE_EXT_FLASH_CONTENT.some((ext) => {
		return (file.name.toLowerCase().endsWith('.' + ext) || file.name.toLowerCase().endsWith('.' + ext + '.unknown'));
	});
};

const is360Image = (file: Misskey.entities.DriveFile): boolean => {
	// First, look for hints in filename/comment
	const text = `${file.name} ${file.comment ?? ''}`.toLowerCase();
	const has360Keyword = Boolean(text.match(/\b(360|pano|panorama|equirect|equirectangular)\b/));

	let w = Number(file.properties.width);
	let h = Number(file.properties.height);
	if (!w || !h) return false;

	// Account for EXIF orientation (same logic as PhotoSwipe)
	if (file.properties.orientation != null && file.properties.orientation >= 5) {
		[w, h] = [h, w];
	}

	// Avoid mistakenly treating portrait photos as panoramas.
	if (w < h) return false;

	// Equirectangular panoramas are typically close to 2:1 aspect ratio
	const ratio = w / h;
	const isLikelyPanorama = w >= 1000 && ratio >= 1.8 && ratio <= 2.5;

	// Use keyword hints only if the dimensions also look like a panorama.
	return isLikelyPanorama && (has360Keyword || ratio >= 1.95);
};

let viewerOverlay: HTMLElement | null = null;
let viewer: any | null = null;

const close360Viewer = () => {
	if (viewer) {
		try {
			viewer.destroy();
		} catch (_err) {
			// ignore
		}
		viewer = null;
	}
	if (viewerOverlay) {
		viewerOverlay.remove();
		viewerOverlay = null;
	}
};

const open360Viewer = async (src: string) => {
	if (!src || viewerOverlay) return;
	const { Viewer } = await import('@photo-sphere-viewer/core');

	viewerOverlay = window.document.createElement('div');
	viewerOverlay.className = 'mk-360-viewer-overlay';
	viewerOverlay.innerHTML = `
		<div class="mk-360-viewer-container"></div>
		<button class="mk-360-viewer-close" type="button" aria-label="Close 360 viewer">×</button>
	`;
	window.document.body.appendChild(viewerOverlay);

	const container = viewerOverlay.querySelector<HTMLDivElement>('.mk-360-viewer-container');
	const closeBtn = viewerOverlay.querySelector<HTMLButtonElement>('.mk-360-viewer-close');

	const close = () => close360Viewer();
	viewerOverlay.addEventListener('click', (e) => {
		if (e.target === viewerOverlay) close();
	});
	closeBtn?.addEventListener('click', close);

	viewer = new Viewer({
		container: container ?? window.document.createElement('div'),
		panorama: src,
		navbar: false,
		defaultLong: 0,
		mousewheel: false,
	});
};

onMounted(() => {
	calcAspectRatio();

	lightbox = new PhotoSwipeLightbox({
		dataSource: props.mediaList
			.filter(media => {
				if (media.type === 'image/svg+xml') return true; // svgのwebpublicはpngなのでtrue
				return media.type.startsWith('image') && FILE_TYPE_BROWSERSAFE.includes(media.type);
			})
			.map(media => {
				const item = {
					src: media.url,
					w: media.properties.width,
					h: media.properties.height,
					alt: media.comment ?? media.name,
					comment: media.comment ?? media.name,
				};
				if (media.properties.orientation != null && media.properties.orientation >= 5) {
					[item.w, item.h] = [item.h, item.w];
				}
				return item;
			}),
		gallery: gallery.value,
		mainClass: 'pswp',
		children: '.image',
		thumbSelector: '.image',
		loop: false,
		padding: window.innerWidth > 500 ? {
			top: 32,
			bottom: 90,
			left: 32,
			right: 32,
		} : {
			top: 0,
			bottom: 78,
			left: 0,
			right: 0,
		},
		imageClickAction: 'close',
		tapAction: 'close',
		bgOpacity: 1,
		showAnimationDuration: 100,
		hideAnimationDuration: 100,
		returnFocus: false,
		pswpModule: PhotoSwipe,
	});

	lightbox.addFilter('itemData', (itemData) => {
		// element is children
		const { element } = itemData;

		const id = element?.dataset.id;
		const file = props.mediaList.find(media => media.id === id);
		if (!file) return itemData;

		itemData.src = file.url;
		itemData.w = Number(file.properties.width);
		itemData.h = Number(file.properties.height);
		if (file.properties.orientation != null && file.properties.orientation >= 5) {
			[itemData.w, itemData.h] = [itemData.h, itemData.w];
		}
		itemData.msrc = file.thumbnailUrl ?? undefined;
		itemData.alt = file.comment ?? undefined;
		itemData.comment = file.comment;
		itemData.thumbCropped = true;
		itemData.is360 = is360Image(file);

		return itemData;
	});

	lightbox.on('uiRegister', () => {
		lightbox?.pswp?.ui?.registerElement({
			name: 'altText',
			className: 'pswp__alt-text-container',
			appendTo: 'wrapper',
			onInit: (el, pswp) => {
				const textBox = window.document.createElement('p');
				textBox.className = 'pswp__alt-text _acrylic';
				el.appendChild(textBox);

				pswp.on('change', () => {
					if (pswp.currSlide?.data.comment) {
						textBox.style.display = '';
					} else {
						textBox.style.display = 'none';
					}

					textBox.textContent = pswp.currSlide?.data.comment;
				});

				// `passive: true` is for Safari compatibility, apparently
				const stopEvent = name => textBox.addEventListener(name, event => event.stopPropagation(), { passive: true });
				stopEvent('wheel');
				stopEvent('pointerdown');
				stopEvent('pointercancel');
			},
		});

		lightbox?.pswp?.ui?.registerElement({
			name: 'view360',
			className: 'pswp__view360-button',
			appendTo: 'bar',
			order: 9,
			isButton: true,
			html: '360°',
			onInit: (el, pswp) => {
				el.setAttribute('aria-label', 'View as 360° Image');

				// Start hidden until we know this is a 360° image.
				const htmlEl = el as HTMLElement;
				htmlEl.style.display = 'none';

				const getCurrentFile = () => {
					const src = pswp.currSlide?.data.src;
					if (!src) return null;
					return props.mediaList.find((m) => m.url === src || m.thumbnailUrl === src || (m.url && src.startsWith(m.url)));
				};

				const updateVisible = () => {
					const is360 = pswp.currSlide?.data.is360;
					if (is360) {
						htmlEl.style.display = 'flex';
					} else {
						htmlEl.style.display = 'none';
					}
				};

				pswp.on('change', updateVisible);
				pswp.on('afterInit', updateVisible);
				updateVisible();

				el.addEventListener('click', (ev) => {
					ev.stopPropagation();
					const file = getCurrentFile();
					const src = file ? file.url : pswp.currSlide?.data.src;
					if (typeof src === 'string' && file && is360Image(file)) {
						open360Viewer(src);
					}
				});
			},
		});
	});

	lightbox.on('afterInit', () => {
		activeEl = window.document.activeElement instanceof HTMLElement ? window.document.activeElement : null;
		focusParent(activeEl, true, true);
		lightbox?.pswp?.element?.focus({
			preventScroll: true,
		});
		window.history.pushState(null, '', '#pswp');
	});

	lightbox.on('destroy', () => {
		close360Viewer();
		focusParent(activeEl, true, false);
		activeEl = null;
		if (window.location.hash === '#pswp') {
			window.history.back();
		}
	});

	window.addEventListener('popstate', popstateHandler);

	lightbox.init();
});

onUnmounted(() => {
	window.removeEventListener('popstate', popstateHandler);
	close360Viewer();
	lightbox?.destroy();
	lightbox = null;
	activeEl = null;
});

const previewable = (file: Misskey.entities.DriveFile): boolean => {
	if (file.type === 'image/svg+xml') return true; // svgのwebpublic/thumbnailはpngなのでtrue
	// FILE_TYPE_BROWSERSAFEに適合しないものはブラウザで表示するのに不適切
	if (isModule(file)) return true;
	if (isFlash(file)) return true;
	return (file.type.startsWith('video') || file.type.startsWith('image')) && FILE_TYPE_BROWSERSAFE.includes(file.type);
};

const openGallery = () => {
	if (props.mediaList.filter(media => previewable(media)).length > 0) {
		lightbox?.loadAndOpen(0);
	}
};

defineExpose({
	openGallery,
});
</script>

<style lang="scss" module>
.root {
	cursor: auto; /* not clickToOpen-able */
}

.container {
	position: relative;
	width: 100%;
}

.medias {
	display: grid;
	grid-gap: 8px;

	height: 100%;
	width: 100%;

	&.n1 {
		grid-template-rows: 1fr;

		// default but fallback (expand)
		min-height: 64px;
		max-height: clamp(
			64px,
			50cqh,
			min(360px, 50vh)
		);

		&.n116_9 {
			min-height: initial;
			max-height: initial;
			aspect-ratio: 16 / 9; // fallback
		}

		&.n11_1{
			min-height: initial;
			max-height: initial;
			aspect-ratio: 1 / 1; // fallback
		}

		&.n12_3 {
			min-height: initial;
			max-height: initial;
			aspect-ratio: 2 / 3; // fallback
		}
	}

	&.n2 {
		aspect-ratio: 16/9;
		grid-template-columns: 1fr 1fr;
		grid-template-rows: 1fr;
	}

	&.n3 {
		aspect-ratio: 16/9;
		grid-template-columns: 1fr 0.5fr;
		grid-template-rows: 1fr 1fr;

		> .media:nth-child(1) {
			grid-row: 1 / 3;
		}

		> .media:nth-child(3) {
			grid-column: 2 / 3;
			grid-row: 2 / 3;
		}
	}

	&.n4 {
		aspect-ratio: 16/9;
		grid-template-columns: 1fr 1fr;
		grid-template-rows: 1fr 1fr;
	}

	&.nMany {
		grid-template-columns: 1fr 1fr;

		> .media {
			aspect-ratio: 16/9;
		}
	}
}

.media {
	overflow: hidden; // clipにするとバグる
	border-radius: var(--MI-radius-sm);
}

:global(.pswp) {
	--pswp-root-z-index: var(--mk-pswp-root-z-index, 2000700) !important;
	--pswp-bg: var(--MI_THEME-modalBg) !important;
}
</style>

<style lang="scss">
.pswp__bg {
	background: var(--MI_THEME-modalBg);
	backdrop-filter: var(--MI-modalBgFilter);
}

.pswp__alt-text-container {
	display: flex;
	flex-direction: row;
	align-items: center;

	position: absolute;
	bottom: 20px;
	left: 50%;
	transform: translateX(-50%);

	width: 75%;
	max-width: 800px;
}

.pswp__alt-text {
	color: var(--MI_THEME-fg);
	margin: 0 auto;
	text-align: center;
	padding: var(--MI-margin);
	border-radius: var(--MI-radius);
	max-height: 8em;
	overflow-y: auto;
	text-shadow: var(--MI_THEME-bg) 0 0 10px, var(--MI_THEME-bg) 0 0 3px, var(--MI_THEME-bg) 0 0 3px;
	white-space: pre-line;
}

.pswp__view360-button {
	display: flex;
	justify-content: center;
	align-self: center;
	width: 38px;
	height: 38px;
	border: none;
	color: #fff;
	font-weight: 700;
	font-size: 12px;
	cursor: pointer;
	padding: 11px 0 0 !important;
	line-height: 38px;
	margin: 0px;
	box-shadow: 0 0 0 1px rgba(255,255,255,0.14);
}

.mk-360-viewer-overlay {
	position: fixed;
	inset: 0;
	z-index: 2147483647;
	background: rgba(0, 0, 0, 0.9);
	display: flex;
	align-items: center;
	justify-content: center;
	padding: 40px;
}

.mk-360-viewer-container {
	width: 100%;
	height: 100%;
	max-width: calc(100vw - 80px);
	max-height: calc(100vh - 80px);
	border-radius: 12px;
	overflow: hidden;
	position: relative;
}

.mk-360-viewer-close {
	position: absolute;
	top: 12px;
	right: 12px;
	width: 36px;
	height: 36px;
	border-radius: 999px;
	border: none;
	background: rgba(0, 0, 0, 0.8);
	color: #fff;
	font-size: 20px;
	line-height: 1;
	cursor: pointer;
}
</style>
