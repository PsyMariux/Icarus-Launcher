<template>
	<div class="popout-menu-trigger" :class="dropdownClass">
		<button
			ref="trigger"
			v-bind="$attrs"
			v-tooltip="tooltip"
			:disabled="disabled"
			:aria-controls="dropdownId"
			:aria-expanded="isOpen"
			aria-haspopup="menu"
			@click="toggle"
			@keydown="handleTriggerKeydown"
		>
			<slot></slot>
		</button>
		<Teleport to="body">
			<div
				v-if="isOpen"
				:id="dropdownId"
				ref="menu"
				class="popout-menu"
				:style="menuStyle"
				role="menu"
				@keydown="handleMenuKeydown"
				@mousedown.stop
			>
				<button class="dummy-button" @focusin="hideAndFocusTrigger"></button>
				<slot name="menu"></slot>
				<button class="dummy-button" @focusin="hideAndFocusTrigger"></button>
			</div>
		</Teleport>
	</div>
</template>

<script setup lang="ts">
import { onClickOutside } from '@vueuse/core'
import { nextTick, onMounted, onUnmounted, ref } from 'vue'

const VIEWPORT_MARGIN = 8

const props = withDefaults(
	defineProps<{
		disabled?: boolean
		dropdownId?: string
		dropdownClass?: string
		tooltip?: string
		placement?: string
	}>(),
	{
		disabled: false,
		dropdownId: undefined,
		dropdownClass: undefined,
		tooltip: undefined,
		placement: 'bottom-end',
	},
)

const emit = defineEmits<{
	open: []
	close: []
}>()

defineOptions({
	inheritAttrs: false,
})

const trigger = ref<HTMLElement | null>(null)
const menu = ref<HTMLElement | null>(null)
const isOpen = ref(false)
const menuStyle = ref({
	top: '0px',
	left: '0px',
	visibility: 'hidden',
})

function clamp(value: number, minimum: number, maximum: number): number {
	return Math.min(Math.max(value, minimum), maximum)
}

function calculatePosition() {
	if (!trigger.value || !menu.value) return

	const triggerRect = trigger.value.getBoundingClientRect()
	const menuRect = menu.value.getBoundingClientRect()
	const [preferredSide, alignment = 'center'] = props.placement.split('-')

	let side = preferredSide
	if (
		side === 'bottom' &&
		triggerRect.bottom + VIEWPORT_MARGIN + menuRect.height > window.innerHeight
	) {
		side = 'top'
	} else if (side === 'top' && triggerRect.top - VIEWPORT_MARGIN - menuRect.height < 0) {
		side = 'bottom'
	} else if (
		side === 'right' &&
		triggerRect.right + VIEWPORT_MARGIN + menuRect.width > window.innerWidth
	) {
		side = 'left'
	} else if (side === 'left' && triggerRect.left - VIEWPORT_MARGIN - menuRect.width < 0) {
		side = 'right'
	}

	let top: number
	let left: number

	if (side === 'left' || side === 'right') {
		left =
			side === 'right'
				? triggerRect.right + VIEWPORT_MARGIN
				: triggerRect.left - menuRect.width - VIEWPORT_MARGIN
		top =
			alignment === 'start'
				? triggerRect.top
				: alignment === 'end'
					? triggerRect.bottom - menuRect.height
					: triggerRect.top + (triggerRect.height - menuRect.height) / 2
	} else {
		top =
			side === 'top'
				? triggerRect.top - menuRect.height - VIEWPORT_MARGIN
				: triggerRect.bottom + VIEWPORT_MARGIN
		left =
			alignment === 'start'
				? triggerRect.left
				: alignment === 'end'
					? triggerRect.right - menuRect.width
					: triggerRect.left + (triggerRect.width - menuRect.width) / 2
	}

	menuStyle.value = {
		top: `${clamp(
			top,
			VIEWPORT_MARGIN,
			Math.max(VIEWPORT_MARGIN, window.innerHeight - menuRect.height - VIEWPORT_MARGIN),
		)}px`,
		left: `${clamp(
			left,
			VIEWPORT_MARGIN,
			Math.max(VIEWPORT_MARGIN, window.innerWidth - menuRect.width - VIEWPORT_MARGIN),
		)}px`,
		visibility: 'visible',
	}
}

async function show() {
	if (props.disabled || isOpen.value) return

	menuStyle.value.visibility = 'hidden'
	isOpen.value = true
	emit('open')
	await nextTick()
	calculatePosition()
}

function hide() {
	if (!isOpen.value) return

	isOpen.value = false
	emit('close')
}

function toggle() {
	if (isOpen.value) {
		hide()
	} else {
		show()
	}
}

function hideAndFocusTrigger() {
	hide()
	trigger.value?.focus()
}

function handleTriggerKeydown(event: KeyboardEvent) {
	if (event.key === 'Enter' || event.key === ' ' || event.key === 'ArrowDown') {
		event.preventDefault()
		show()
	} else if (event.key === 'Escape') {
		event.preventDefault()
		hide()
	}
}

function handleMenuKeydown(event: KeyboardEvent) {
	if (event.key === 'Escape') {
		event.preventDefault()
		hideAndFocusTrigger()
	}
}

function updatePosition() {
	if (isOpen.value) {
		calculatePosition()
	}
}

onClickOutside(menu, hide, { ignore: [trigger] })

onMounted(() => {
	window.addEventListener('resize', updatePosition)
	window.addEventListener('scroll', updatePosition, true)
})

onUnmounted(() => {
	window.removeEventListener('resize', updatePosition)
	window.removeEventListener('scroll', updatePosition, true)
})

defineExpose({
	show,
	hide,
})
</script>

<style scoped>
.popout-menu-trigger {
	display: inline-block;
	position: relative;
}

.popout-menu {
	position: fixed;
	z-index: 9999;
	width: fit-content;
	max-width: calc(100vw - 16px);
	max-height: calc(100vh - 16px);
	overflow: auto;
	border: 1px solid var(--color-divider);
	padding: var(--gap-sm);
	border-radius: var(--radius-md);
	background-color: var(--color-raised-bg);
	box-shadow: var(--shadow-floating);
}

.dummy-button {
	position: absolute;
	width: 0;
	height: 0;
	margin: 0;
	padding: 0;
	border: none;
	overflow: hidden;
	clip: rect(0 0 0 0);
	white-space: nowrap;
	outline: none;
}
</style>
