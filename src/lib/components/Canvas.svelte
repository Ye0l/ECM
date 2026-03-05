<script lang="ts">
	import type { Snippet } from 'svelte';
	import { setContext } from 'svelte';

	let { 
		children, 
		oncanvasdoubleclick,
		panX = $bindable(0),
		panY = $bindable(0),
		scale = $bindable(1)
	} = $props<{
		children?: Snippet;
		oncanvasdoubleclick?: (x: number, y: number) => void;
		panX?: number;
		panY?: number;
		scale?: number;
	}>();

	// 하위 컴포넌트(Node 등)에서 스케일에 접근할 수 있도록 컨텍스트 제공
	setContext('canvas', {
		get scale() { return scale; }
	});

	let isDragging = $state(false);
	let startX = 0;
	let startY = 0;

	function handlePointerDown(e: PointerEvent) {
		// 노드 등 다른 요소에서 발생한 이벤트는 무시
		if (e.target !== e.currentTarget) return;
		if (e.button !== 0 && e.button !== 1) return; // 좌클릭 또는 휠클릭

		(e.currentTarget as HTMLElement)?.setPointerCapture(e.pointerId);
		isDragging = true;
		startX = e.clientX - panX;
		startY = e.clientY - panY;
	}

	function handlePointerMove(e: PointerEvent) {
		if (!isDragging) return;
		panX = e.clientX - startX;
		panY = e.clientY - startY;
	}

	function handlePointerUp(e: PointerEvent) {
		if (!isDragging) return;
		isDragging = false;
		(e.currentTarget as HTMLElement)?.releasePointerCapture(e.pointerId);
	}

	function handleWheel(e: WheelEvent) {
		e.preventDefault(); // 스크롤 방지 공통 처리

		// Shift 키 누르고 스크롤하면 보통 좌우로 이동
		if (e.ctrlKey) {
			// Ctrl + Wheel: 줌
		} else if (e.target === e.currentTarget) {
			// 그냥 화면 스크롤인 경우지만 줌으로 통일
		}
		
		const zoomSensitivity = 0.001;
		const delta = -e.deltaY * zoomSensitivity;
		
		// 스케일 제한 (10% ~ 500%)
		const newScale = Math.min(Math.max(0.1, scale + delta * scale), 5);

		if (newScale !== scale) {
			// 마우스 포인터 기준으로 줌
			const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
			const cursorX = e.clientX - rect.left;
			const cursorY = e.clientY - rect.top;

			panX = cursorX - (cursorX - panX) * (newScale / scale);
			panY = cursorY - (cursorY - panY) * (newScale / scale);
			
			scale = newScale;
		}
	}

	function handleDoubleClick(e: MouseEvent) {
		if (e.target !== e.currentTarget) return; // 빈 캔버스에서만 더블클릭 처리
		if (oncanvasdoubleclick) {
			// 화면 좌표를 캔버스 논리 좌표로 변환
			const rect = (e.currentTarget as HTMLElement).getBoundingClientRect();
			const cursorX = e.clientX - rect.left;
			const cursorY = e.clientY - rect.top;
			
			const logicalX = (cursorX - panX) / scale;
			const logicalY = (cursorY - panY) / scale;
			
			oncanvasdoubleclick(logicalX, logicalY);
		}
	}
</script>

<div
	role="presentation"
	class="canvas-container"
	onpointerdown={handlePointerDown}
	onpointermove={handlePointerMove}
	onpointerup={handlePointerUp}
	onpointercancel={handlePointerUp}
	onwheel={handleWheel}
	ondblclick={handleDoubleClick}
	style="--pan-x: {panX}; --pan-y: {panY}; --scale: {scale};"
>
	<div
		class="canvas-content"
		style="transform: translate({panX}px, {panY}px) scale({scale});"
	>
		{@render children?.()}
	</div>
</div>

<style>
	.canvas-container {
		width: 100vw;
		height: 100vh;
		overflow: hidden;
		position: relative;
		cursor: grab;
		background-color: #1e1e1e;
		
		/* 줌/팬에 반응하는 도트 배경 */
		background-image: radial-gradient(circle, rgba(255,255,255,0.15) calc(1.5px * var(--scale)), transparent calc(1px * var(--scale)));
		background-size: calc(20px * var(--scale)) calc(20px * var(--scale));
		background-position: calc(var(--pan-x) * 1px) calc(var(--pan-y) * 1px);
	}
	
	.canvas-container:active {
		cursor: grabbing;
	}

	.canvas-content {
		transform-origin: 0 0;
		position: absolute;
		top: 0;
		left: 0;
		width: 0;
		height: 0;
	}
</style>
