<script setup lang="ts">
import { ref } from 'vue';
import { Reposition } from '@/quick/reposition';
import { Singleton } from '@/lib/renderer/singleton';


const TOUCH_DELAY = 500;

const INNER_HTML_KEYWORD = '{InnerHTML}';
const VALUE_KEYWORD = '{Value}';

const SELECTED_KEYWORD = 'selected';
const ATTRIBUTE_KEYWORD = 'tooltip';
const ATTRIBUTE_SELECTED_KEYWORD = 'selected-tooltip';
const IGNORE_ELLIPSES_KEYWORD = 'tooltip-ignore-ellipses';


const $text = ref<string>('');
const $visible = ref<boolean>(false);
const $left = ref<number>(0);
const $top = ref<number>(0);
const $target = ref<HTMLElement>(null);
const $label_node = ref<HTMLElement>();
const $touch_timeout = ref<NodeJS.Timeout>(null);

function Move_Internal(x0: number, y0: number, rect: DOMRect) {
	let {x, y} =
		Reposition(
			x0,
			y0,
			rect
		);
	
	$left.value = x;
	$top.value = y;
}

function Move_Touch(event: TouchEvent) {
	let {
		clientX,
		clientY,
		radiusY
	} = event.touches[0];

	let rect =
		$label_node.value
		.getBoundingClientRect();
	
	Move_Internal(
		clientX - rect.width * 0.5,
		clientY - rect.height - radiusY * 0.5 - 32,
		rect
	);
}

function Move_Mouse(event: MouseEvent) {
	let {
		clientX,
		clientY
	} = event;

	let rect =
		$label_node.value
		.getBoundingClientRect();
	
	Move_Internal(
		clientX,
		clientY - rect.height,
		rect
	);
}

function GetText_Internal(node: HTMLElement) {
	if (node.hasAttribute(SELECTED_KEYWORD) &&
		node.hasAttribute(ATTRIBUTE_SELECTED_KEYWORD))
		return node.getAttribute(ATTRIBUTE_SELECTED_KEYWORD);

	if (node.hasAttribute(ATTRIBUTE_KEYWORD))
		return node.getAttribute(ATTRIBUTE_KEYWORD);

	if (!node.childElementCount &&
		node.childNodes.length &&
		!node.hasAttribute(IGNORE_ELLIPSES_KEYWORD) &&
		node.offsetWidth < node.scrollWidth)
		return node.innerHTML;

	return null;
}

function GetText(node: HTMLElement) {
	let text = GetText_Internal(node);

	if (!text)
		return null;

	text = text.replaceAll(INNER_HTML_KEYWORD, node.innerHTML);

	if (node instanceof HTMLInputElement)
		text = text.replaceAll(VALUE_KEYWORD, node.value);

	return text;
}


function Show(next_target: HTMLElement, text: string) {
	$target.value = next_target;
	$label_node.value.innerHTML = text;
	$visible.value = true;
}

function Hide() {
	$visible.value = false;
}


function OnTouchStart(event: TouchEvent) {
	if ($touch_timeout.value != null) {
		clearTimeout($touch_timeout.value);
		$touch_timeout.value = null;
	}

	function Timeout() {
		event.stopPropagation();

		$touch_timeout.value = null;

		let target0 = event.target as HTMLElement;
		let text0 = GetText(target0);

		if (!$text.value)
			return;

		Show(target0, text0);
		Move_Touch(event);
	}
	
	$touch_timeout.value = setTimeout(Timeout, TOUCH_DELAY);
}

function OnTouchEnd(event: TouchEvent) {
	$target.value = null;
	$visible.value = false;
	

	if ($touch_timeout.value == null)
		return;

	clearTimeout($touch_timeout.value);

	$touch_timeout.value = null;
}

function OnTouchMove(event: TouchEvent) {
	if ($touch_timeout.value == null)
		return;

	clearTimeout($touch_timeout.value);

	$touch_timeout.value = null;
}


function OnPointerMove(event: PointerEvent) {
	if (event.pointerType !== 'mouse')
		return;

	if ($target.value != event.target) {
		$target.value = null;

		let target0 = event.target as HTMLElement;
		let text = GetText(target0);

		if (!text)
			return Hide();

		Show(target0, text);
	}

	Move_Mouse(event);
}

function OnPointerDown(event: PointerEvent) {
	Hide();
}


defineExpose({
	$text,
	$visible
});

defineOptions(Singleton({
	name: 'Tooltip'
}));

window
.addEventListener('pointermove', OnPointerMove);
window
.addEventListener('pointerdown', OnPointerDown);
window
.addEventListener('touchstart', OnTouchStart);
window
.addEventListener('touchend', OnTouchEnd);
window
.addEventListener('touchmove', OnTouchMove);
</script>

<template>
  <div
    class="root absolute top-0 left-0 w-full h-full"
  >
    <label
      ref="$label_node"
      class="window absolute flex flex-fixed flex-col -white px-[18px] py-[6px] font-medium
			border-[1px] border-black shadow-xl shrinkable overflow-y-auto max-w-[256px] text-center"
      :style="`left: ${$left}px; top: ${$top}px`"
      :visible="$visible"
    >{{ $text }}</label>
  </div>
</template>

<style scoped>
@import '@/styles/size.css';
@import '@/styles/flex.css';
@import '@/styles/behavior.css';

@tailwind base;
@tailwind components;
@tailwind utilities;

.root[visible="false"],
.root[visible="false"] * {
	pointer-events: none !important;
}

.window[visible="false"] {
	@apply shrink;
}

</style>
