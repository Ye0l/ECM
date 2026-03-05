<script lang="ts">
	import { getContext } from 'svelte';
	import { marked } from 'marked';
	import type { AccountEntry } from './NodeEditor.svelte';

	export type PortPosition = 'top' | 'right' | 'bottom' | 'left';

	let { id, x, y, pinned = false, text, entries = [], ondragstart, ondrag, ondragend, onpin, onedit, onportdown, onportenter, onresize } = $props<{
		id: string;
		x: number;
		y: number;
		pinned?: boolean;
		text: string;
		entries?: AccountEntry[];
		ondragstart: (id: string) => void;
		ondrag: (id: string, newX: number, newY: number) => void;
		ondragend: (id: string) => void;
		onpin: (id: string, isPinned: boolean) => void;
		onedit: () => void;
		onportdown?: (e: PointerEvent, nodeId: string, port: PortPosition) => void;
		onportenter?: (nodeId: string, port: PortPosition) => void;
		onresize?: (nodeId: string, width: number, height: number) => void;
	}>();

	const canvas = getContext<{ scale: number }>('canvas');

	let isDragging = false;
	let startX = 0;
	let startY = 0;
	
	// 외부 x, y 기반 렌더링을 위한 임시 캐시. (드래그 중에는 부드러운 이동을 위해 로컬 값 사용)
	// svelte-ignore state_referenced_locally
	let localX = $state(x);
	// svelte-ignore state_referenced_locally
	let localY = $state(y);

	// 마크다운과 커스텀 위키 링크 파싱 (`[[링크 대상]]`)
	const htmlContent = $derived.by(() => {
		// 우선 `[[링크 텍스트]]`를 span 클래스로 치환 (추후 SVG 선 연결에 참고 가능)
		const linkedText = text.replace(/\[\[(.*?)\]\]/g, '<span class="wiki-link">#$1</span>');
		return marked(linkedText);
	});

	let nodeWidth = $state(250);
	let nodeHeight = $state(120);

	$effect(() => {
		// 외부 시뮬레이션에서 x, y 값이 변경될 때 로컬 값도 추적 (직접 드래그 중이 아닐 때)
		if (!isDragging) {
			localX = x;
			localY = y;
		}
	});

	$effect(() => {
		onresize?.(id, nodeWidth, nodeHeight);
	});

	function handlePointerDown(e: PointerEvent) {
		if (e.button !== 0) return;
		isDragging = true;
		e.stopPropagation();
		(e.currentTarget as HTMLElement)?.setPointerCapture(e.pointerId);
		startX = e.clientX;
		startY = e.clientY;
		localX = x;
		localY = y;
		ondragstart(id);
	}

	function handlePointerMove(e: PointerEvent) {
		if (!isDragging) return;
		e.stopPropagation();

		const dx = (e.clientX - startX) / Math.max(0.1, canvas.scale); // scale이 0에 수렴 방지
		const dy = (e.clientY - startY) / Math.max(0.1, canvas.scale);

		localX += dx;
		localY += dy;

		startX = e.clientX;
		startY = e.clientY;

		ondrag(id, localX, localY);
	}

	function handlePointerUp(e: PointerEvent) {
		if (!isDragging) return;
		isDragging = false;
		e.stopPropagation();
		(e.currentTarget as HTMLElement)?.releasePointerCapture(e.pointerId);
		ondragend(id);
	}

	function handlePinClick(e: MouseEvent) {
		e.stopPropagation();
		onpin(id, !pinned);
	}

	function handleDoubleClick(e: MouseEvent) {
		e.stopPropagation();
		onedit();
	}

	function handlePortDown(e: PointerEvent, port: PortPosition) {
		e.stopPropagation(); // 노드 드래그 방지
		onportdown?.(e, id, port);
	}

	function handlePortEnter(port: PortPosition) {
		onportenter?.(id, port);
	}

	// 수입/지출/분류 포맷터
	function formatAmount(amt: number) {
		return new Intl.NumberFormat('ko-KR').format(amt) + '원';
	}
	
	const leftEntries = $derived(entries.filter((e: AccountEntry) => e.side === 'left'));
	const rightEntries = $derived(entries.filter((e: AccountEntry) => e.side === 'right'));
</script>

<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
<div
	role="presentation"
	class="node"
	class:has-entries={entries && entries.length > 0}
	class:pinned={pinned}
	style="transform: translate({localX}px, {localY}px);"
	bind:clientWidth={nodeWidth}
	bind:clientHeight={nodeHeight}
	onpointerdown={handlePointerDown}
	onpointermove={handlePointerMove}
	onpointerup={handlePointerUp}
	onpointercancel={handlePointerUp}
	ondblclick={handleDoubleClick}
>
	<!-- 포트 노드들 -->
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="port top" 
		data-nodeid={id} data-port="top"
		onpointerdown={(e) => handlePortDown(e, 'top')} 
		onpointerenter={() => handlePortEnter('top')}
	></div>
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="port right" 
		data-nodeid={id} data-port="right"
		onpointerdown={(e) => handlePortDown(e, 'right')} 
		onpointerenter={() => handlePortEnter('right')}
	></div>
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="port bottom" 
		data-nodeid={id} data-port="bottom"
		onpointerdown={(e) => handlePortDown(e, 'bottom')} 
		onpointerenter={() => handlePortEnter('bottom')}
	></div>
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="port left" 
		data-nodeid={id} data-port="left"
		onpointerdown={(e) => handlePortDown(e, 'left')} 
		onpointerenter={() => handlePortEnter('left')}
	></div>

	<!-- 고정(Pin) 토글 버튼 추가 -->
	<button class="pin-btn" class:active={pinned} onclick={handlePinClick} onpointerdown={(e) => e.stopPropagation()}>
		📌
	</button>

	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="node-content markdown-body">
		{@html htmlContent}
	</div>
	
	<!-- 가계부 정보 영역: 복식부기(차변/대변 뷰) -->
	{#if entries && entries.length > 0}
		<div class="node-entries-info">
			{#if leftEntries.length > 0}
				<div class="side left">
					{#each leftEntries as entry}
						<div class="entry-item">
							<span class="acc-badge">{entry.accountName}</span>
							<span class="amount">{formatAmount(entry.amount)}</span>
						</div>
					{/each}
				</div>
			{/if}
			
			{#if leftEntries.length > 0 && rightEntries.length > 0}
				<div class="divider"></div>
			{/if}

			{#if rightEntries.length > 0}
				<div class="side right">
					{#each rightEntries as entry}
						<div class="entry-item">
							<span class="acc-badge">{entry.accountName}</span>
							<span class="amount">{formatAmount(entry.amount)}</span>
						</div>
					{/each}
				</div>
			{/if}
		</div>
	{/if}
</div>

<style>
	.node {
		position: absolute;
		top: 0;
		left: 0;
		width: 250px;
		background-color: #2d2d2d;
		border: 1px solid #4d4d4d;
		border-radius: 8px;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
		cursor: grab;
		user-select: none;
		touch-action: none;
		transition: box-shadow 0.2s, border-color 0.2s, background-color 0.2s;
	}
	.node:active {
		cursor: grabbing;
		box-shadow: 0 8px 24px rgba(0, 0, 0, 0.5);
		z-index: 10;
	}
	
	/* 항목 포함 시 외형 조정 */
	.node.has-entries {
		border-color: #555555; 
	}
	
	/* Pinned 상태일 때 외곽선 효과 */
	.node.pinned {
		box-shadow: 0 0 0 2px rgba(255, 215, 0, 0.6), 0 4px 12px rgba(0, 0, 0, 0.5);
	}

	/* Port Styling */
	.port {
		position: absolute;
		width: 12px;
		height: 12px;
		background-color: #444;
		border: 2px solid #2d2d2d;
		border-radius: 50%;
		z-index: 15;
		cursor: crosshair;
		transition: background-color 0.2s, transform 0.1s;
	}
	.port:hover {
		background-color: #4da6ff;
		transform: scale(1.3);
	}
	
	/* Port Positions */
	.port.top {
		top: -6px;
		left: 50%;
		transform: translateX(-50%);
	}
	.port.top:hover {
		transform: translateX(-50%) scale(1.3);
	}

	.port.bottom {
		bottom: -6px;
		left: 50%;
		transform: translateX(-50%);
	}
	.port.bottom:hover {
		transform: translateX(-50%) scale(1.3);
	}

	.port.left {
		top: 50%;
		left: -6px;
		transform: translateY(-50%);
	}
	.port.left:hover {
		transform: translateY(-50%) scale(1.3);
	}

	.port.right {
		top: 50%;
		right: -6px;
		transform: translateY(-50%);
	}
	.port.right:hover {
		transform: translateY(-50%) scale(1.3);
	}


	.node-content {
		padding: 16px;
		color: #e0e0e0;
		font-size: 14px;
		line-height: 1.5;
		word-break: break-all;
	}
	
	/* 하단 가계부 정보 영역 */
	.node-entries-info {
		border-top: 1px solid rgba(255, 255, 255, 0.1);
		display: flex;
		background: rgba(0, 0, 0, 0.2);
		border-bottom-left-radius: 8px;
		border-bottom-right-radius: 8px;
		min-height: 48px;
	}
	
	.divider {
		width: 1px;
		background-color: rgba(255, 255, 255, 0.1);
	}
	
	.side {
		flex: 1;
		display: flex;
		flex-direction: column;
		padding: 8px 12px;
		gap: 6px;
	}
	
	.side.left {
		border-bottom-left-radius: 8px;
	}
	.side.right {
		border-bottom-right-radius: 8px;
	}

	.entry-item {
		display: flex;
		justify-content: space-between;
		align-items: center;
		gap: 8px;
	}
	
	.acc-badge {
		font-size: 11px;
		padding: 2px 5px;
		border-radius: 3px;
		font-weight: 500;
		background-color: rgba(255,255,255, 0.1);
		color: #ccc;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		max-width: 80px;
	}
	
	.amount {
		font-size: 13px;
		font-weight: 600;
		color: #ffffff;
	}

	:global(.markdown-body p) {
		margin: 0 0 10px 0;
	}
	:global(.markdown-body p:last-child) {
		margin-bottom: 0;
	}
	:global(.markdown-body .wiki-link) {
		color: #4da6ff;
		font-weight: 600;
		cursor: pointer;
	}
	:global(.markdown-body .wiki-link:hover) {
		text-decoration: underline;
	}
	
	.pin-btn {
		position: absolute;
		top: 8px;
		right: 8px;
		background: none;
		border: none;
		font-size: 16px;
		cursor: pointer;
		opacity: 0.3;
		padding: 4px;
		border-radius: 4px;
		transition: opacity 0.2s, background-color 0.2s;
		z-index: 20;
	}
	.pin-btn:hover {
		opacity: 0.8;
		background-color: rgba(255, 255, 255, 0.1);
	}
	.pin-btn.active {
		opacity: 1;
		background-color: rgba(255, 215, 0, 0.3);
		text-shadow: 0 0 5px rgba(255, 215, 0, 0.8);
	}
	
	/* Pinned 상태일 때 외곽선 효과 */
	.node.pinned {
		box-shadow: 0 0 0 2px rgba(255, 215, 0, 0.6), 0 4px 12px rgba(0, 0, 0, 0.5);
	}
</style>
