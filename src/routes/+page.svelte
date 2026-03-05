<script lang="ts">
	import { onMount, tick } from 'svelte';
	import * as d3 from 'd3';
	import Canvas from '$lib/components/Canvas.svelte';
	import Node from '$lib/components/Node.svelte';
	import NodeEditor from '$lib/components/NodeEditor.svelte';
	import Help from '$lib/components/Help.svelte';
	import Sidebar from '$lib/components/Sidebar.svelte';
	import BottomInput from '$lib/components/BottomInput.svelte';
	import type { AccountEntry } from '$lib/components/NodeEditor.svelte';

	type CanvasInfo = { id: string; name: string };
	
	type Note = { 
		id: string; 
		x: number; 
		y: number; 
		fx?: number | null; 
		fy?: number | null; 
		pinned?: boolean; 
		text: string; 
		width?: number; 
		height?: number; 
		entries?: AccountEntry[]; 
	};
	
	type Link = {
		id: string;
		source: string | Note; 
		target: string | Note;
		value?: number;
	};

	type PortPosition = 'top' | 'right' | 'bottom' | 'left';
	type ManualConnection = {
		id: string;
		source_node_id: string;
		source_port: PortPosition;
		target_node_id: string;
		target_port: PortPosition;
		canvas_id?: string;
	};

	const STORAGE_CANVASES = 'ecm_canvases';
	const STORAGE_NOTES = 'ecm_notes';
	const STORAGE_CONNS = 'ecm_connections';

	function getLS<T>(key: string, _default: T): T {
		if (typeof localStorage === 'undefined') return _default;
		const v = localStorage.getItem(key);
		return v ? JSON.parse(v) : _default;
	}
	function setLS(key: string, val: any) {
		if (typeof localStorage !== 'undefined') localStorage.setItem(key, JSON.stringify(val));
	}
	function genId() { return Math.random().toString(36).substr(2, 9); }

	let canvases = $state<CanvasInfo[]>([]);
	let activeCanvasId = $state<string | null>(null);
	
	let allNotes = $state<Note[]>([]); // DB에서 로드한 전체 원본 노드
	let filterKeyword = $state(''); // 툴바 검색어 필터
	
	let notes = $state<Note[]>([]); // 필터를 거쳐 캔버스/로직에 띄울 실제 시뮬레이션 대상 노드들
	let forceLinks = $state<Link[]>([]);
	
	let isEditing = $state(false);
	let editTargetX = $state(0);
	let editTargetY = $state(0);
	let editTargetId = $state<string | null>(null);
	let editInitialText = $state('');

	let enableAutoLink = $state(true); 
	let manualConns = $state<ManualConnection[]>([]);
	
	let isConnecting = $state(false);
	let connStartNodeId = $state<string | null>(null);
	let connStartPort = $state<PortPosition | null>(null);
	let connCurrentX = $state(0);
	let connCurrentY = $state(0);

	let canvasScale = $state(1);
	let canvasPanX = $state(0);
	let canvasPanY = $state(0);

	// D3 시뮬레이션 인스턴스
	let simulation: d3.Simulation<Note, Link> | null = null;
    // 시뮬레이션 상태 갱신 트리거 (Svelte 5 렌더링용)
	let tickCount = $state(0);

	onMount(async () => {
		await loadCanvases();
	});

	async function loadCanvases() {
		canvases = getLS<CanvasInfo[]>(STORAGE_CANVASES, []);
		if (canvases.length > 0 && !activeCanvasId) {
			activeCanvasId = canvases[0].id;
			await loadNotes(activeCanvasId);
		}
	}

	async function loadNotes(canvasId: string) {
		const rawNotes = getLS<Record<string, Note[]>>(STORAGE_NOTES, {});
		const storedNotes = rawNotes[canvasId] || [];
		
		// 초기 위치 등 정규화하여 allNotes에 캐싱
		allNotes = storedNotes.map(n => ({
			...n,
			fx: n.pinned ? n.x : null,
			fy: n.pinned ? n.y : null
		}));

		const allConns = getLS<ManualConnection[]>(STORAGE_CONNS, []);
		manualConns = allConns.filter(c => c.canvas_id === canvasId);
		
		applyFilterAndRebuild();
	}
	
	function applyFilterAndRebuild() {
		if (!filterKeyword.trim()) {
			notes = allNotes.map(n => ({...n})); // 시뮬레이션 뮤테이션을 허용하는 복사본
		} else {
			const lowerKey = filterKeyword.trim().toLowerCase();
			const matched = allNotes.filter(n => 
				n.text.toLowerCase().includes(lowerKey) || 
				n.entries?.some(e => e.accountName.toLowerCase().includes(lowerKey))
			).map(n => ({...n}));
			
			// 필터 앵커 노드 (가상의 중심 고정 노드)
			const anchorNode: Note = {
				id: 'filter-anchor-node',
				text: `🔍 필터: **${filterKeyword}**`,
				x: window.innerWidth / 2,
				y: window.innerHeight / 2,
				fx: window.innerWidth / 2,
				fy: window.innerHeight / 2,
				pinned: true,
				entries: []
			};
			
			notes = [anchorNode, ...matched];
		}
		initSimulation();
	}

	function rebuildDynamicLinks() {
		if (!enableAutoLink) {
			forceLinks = [];
			return;
		}
		
		const links: Link[] = [];
		const regex = /\[\[(.*?)\]\]/g;
		
		for (const source of notes) {
			regex.lastIndex = 0;
			let match;
			
			// 수동 태그 파싱
			while ((match = regex.exec(source.text)) !== null) {
				const targetText = match[1]?.trim().toLowerCase();
				if (!targetText) continue;
				const target = notes.find(n => n.id !== source.id && n.id !== 'filter-anchor-node' && n.text.toLowerCase().includes(targetText));
				if (target) {
					links.push({
						id: `auto-${source.id}-${target.id}`,
						source: source.id,
						target: target.id
					});
				}
			}
			
			// 복식부기 entry(계정명)를 통한 매칭 연결
			if (source.entries) {
				for (const entry of source.entries) {
					const entryName = entry.accountName.toLowerCase();
					// 계정명이 다른 노드의 텍스트에 포함되거나 서로 같은 계정명을 들고 있으면 연결
					const targets = notes.filter(n => n.id !== source.id && n.id !== 'filter-anchor-node' && (
						n.text.toLowerCase().includes(entryName) ||
						n.entries?.some(e => e.accountName.toLowerCase() === entryName)
					));
					for (const target of targets) {
						const lid = `entry-${source.id}-${target.id}`;
						if (!links.some(l => l.id === lid || l.id === `entry-${target.id}-${source.id}`)) {
							links.push({ id: lid, source: source.id, target: target.id });
						}
					}
				}
			}
			
			// 필터 앵커 노드 자동 연결 (중심으로 모이게 함)
			if (filterKeyword.trim() && source.id !== 'filter-anchor-node') {
				links.push({
					id: `filter-link-${source.id}`,
					source: 'filter-anchor-node',
					target: source.id
				});
			}
		}
		forceLinks = links;
	}

	function initSimulation() {
		if (simulation) simulation.stop();
		
		rebuildDynamicLinks();

		simulation = d3.forceSimulation<Note>(notes)
			.force('charge', d3.forceManyBody().strength(-1000))
			.force('center', d3.forceCenter(400, 300).strength(0.05))
			.force('collision', d3.forceCollide().radius(180)) // 노드 겹침 방지 반경
			.force('link', d3.forceLink<Note, Link>(forceLinks).id(d => d.id).distance(300))
			.on('tick', () => {
				tickCount++; // trigger svelte reactivity
			});
			
		simulation.alpha(1).restart();
	}

	// 노드 목록 혹은 연결선 정책 바뀔 때마다 시뮬레이션 반영
	$effect(() => {
		if (simulation) {
			rebuildDynamicLinks();
			simulation.nodes(notes);
			const linkForce = simulation.force<d3.ForceLink<Note, Link>>('link');
			if (linkForce) linkForce.links(forceLinks);
			simulation.alphaTarget(0.1).restart();
			setTimeout(() => {
			    if (simulation) simulation.alphaTarget(0); // 잠시 후 식히기
			}, 300);
		}
	});

	// localStorage에 저장 (Pined 좌표, 데이터 등)
	function saveCurrentStateToLS() {
		if (!activeCanvasId) return;
		
		// 현재 화면(notes)의 변경된 x,y 위치를 allNotes에 반영
		for (const n of notes) {
			if (n.id === 'filter-anchor-node') continue;
			const idx = allNotes.findIndex(an => an.id === n.id);
			if (idx !== -1) {
				allNotes[idx].x = n.x;
				allNotes[idx].y = n.y;
				allNotes[idx].fx = n.fx;
				allNotes[idx].fy = n.fy;
				allNotes[idx].pinned = n.pinned;
				allNotes[idx].text = n.text;
				allNotes[idx].entries = n.entries;
			} else {
				// 새로 생성된 노트인 경우
				allNotes.push({...n});
			}
		}

		const storageData = getLS<Record<string, Note[]>>(STORAGE_NOTES, {});
		storageData[activeCanvasId] = allNotes.map(n => ({
			id: n.id,
			text: n.text,
			width: n.width,
			height: n.height,
			entries: n.entries,
			pinned: n.pinned,
			x: n.x,
			y: n.y,
			fx: n.pinned ? n.x : null,
			fy: n.pinned ? n.y : null
		}));
		setLS(STORAGE_NOTES, storageData);
	}

	async function handleSelectCanvas(id: string) {
		activeCanvasId = id;
		await loadNotes(id);
	}

	async function handleCreateCanvas(name: string) {
		const newCanvas = { id: genId(), name };
		canvases = [newCanvas, ...canvases];
		setLS(STORAGE_CANVASES, canvases);
		await handleSelectCanvas(newCanvas.id);
	}

	async function handleDeleteCanvas(id: string) {
		if (simulation) simulation.stop();
		canvases = canvases.filter(c => c.id !== id);
		setLS(STORAGE_CANVASES, canvases);

		const allNotes = getLS<Record<string, Note[]>>(STORAGE_NOTES, {});
		delete allNotes[id];
		setLS(STORAGE_NOTES, allNotes);

		const allConns = getLS<ManualConnection[]>(STORAGE_CONNS, []);
		setLS(STORAGE_CONNS, allConns.filter(c => c.canvas_id !== id));

		if (activeCanvasId === id) {
			activeCanvasId = canvases.length > 0 ? canvases[0].id : null;
			if (activeCanvasId) await loadNotes(activeCanvasId);
			else {
				notes = [];
				forceLinks = [];
			}
		}
	}

	function handleBottomInputSave(text: string) {
		handleSaveNote(text, 200, 200); 
	}

	function handleDoubleClick(x: number, y: number) {
		if (!activeCanvasId) return;
		editTargetX = x;
		editTargetY = y;
		editTargetId = null;
		editInitialText = '';
		isEditing = true;
	}

	async function handleSaveNote(text: string, x: number, y: number, entries: AccountEntry[] = []) {
		if (!activeCanvasId) return;
		
		isEditing = false;
		
		if (editTargetId) {
			const note = notes.find(n => n.id === editTargetId);
			if (note) {
				note.text = text;
				note.entries = entries;
			}
			notes = [...notes]; // Svelte $state reactivity trigger
		} else {
			const newNote: Note = {
				id: genId(),
				x, y, text,
				entries,
				pinned: false,
				fx: null, fy: null
			};
			notes = [...notes, newNote];
		}
		
		saveCurrentStateToLS();
		if (simulation) {
			simulation.alpha(0.5).restart();
		}
	}

	function handleCancelEdit() {
		isEditing = false;
	}

	// --- D3 Node Drag Callbacks ---
	function handleDragStart(id: string) {
		if (simulation) simulation.alphaTarget(0.3).restart();
		const note = notes.find(n => n.id === id);
		if (note) {
			note.fx = note.x;
			note.fy = note.y;
		}
	}

	function handleDrag(id: string, newX: number, newY: number) {
		const note = notes.find(n => n.id === id);
		if (note) {
			note.fx = newX;
			note.fy = newY;
		}
	}

	function handleDragEnd(id: string) {
		if (simulation) simulation.alphaTarget(0);
		const note = notes.find(n => n.id === id);
		if (note) {
			if (!note.pinned) {
				note.fx = null;
				note.fy = null;
			}
			saveCurrentStateToLS();
		}
	}

	function handlePinNode(id: string, isPinned: boolean) {
		const note = notes.find(n => n.id === id);
		if (note) {
			note.pinned = isPinned;
			if (isPinned) {
				note.fx = note.x;
				note.fy = note.y;
			} else {
				note.fx = null;
				note.fy = null;
			}
			saveCurrentStateToLS();
			notes = [...notes];
		}
	}
	// ------------------------------

	function handleNodeEdit(id: string, currentText: string, x: number, y: number) {
		editTargetId = id;
		editInitialText = currentText;
		editTargetX = x;
		editTargetY = y;
		isEditing = true;
	}

	function handleNodeResize(id: string, width: number, height: number) {
		const note = notes.find((n) => n.id === id);
		if (note) {
			note.width = width;
			note.height = height;
		}
	}

	function getPortCoords(node: Note, port: PortPosition) {
		const w = node.width || 250;
		const h = node.height || 120;
		switch (port) {
			case 'top': return { x: node.x + w / 2, y: node.y };
			case 'bottom': return { x: node.x + w / 2, y: node.y + h };
			case 'left': return { x: node.x, y: node.y + h / 2 };
			case 'right': return { x: node.x + w, y: node.y + h / 2 };
			default: return { x: node.x, y: node.y };
		}
	}

	function createSCurve(x1: number, y1: number, p1: PortPosition, x2: number, y2: number, p2: PortPosition | null) {
		const offset1 = { x: p1 === 'right' ? 50 : p1 === 'left' ? -50 : 0, y: p1 === 'bottom' ? 50 : p1 === 'top' ? -50 : 0 };
		const offset2 = p2 ? { x: p2 === 'right' ? 50 : p2 === 'left' ? -50 : 0, y: p2 === 'bottom' ? 50 : p2 === 'top' ? -50 : 0 } : { x: 0, y: 0 };
		
		const cp1x = x1 + offset1.x;
		const cp1y = y1 + offset1.y;
		const cp2x = x2 + offset2.x;
		const cp2y = y2 + offset2.y;

		return `M ${x1} ${y1} C ${cp1x} ${cp1y}, ${cp2x} ${cp2y}, ${x2} ${y2}`;
	}

	// 계산된 자동 연결선 (D3 Force용 직선이 아닌 S-Curve 유지 - Node 중심점 기반)
	const drawingAutoConnections = $derived.by(() => {
		/* eslint-disable-next-line @typescript-eslint/no-unused-expressions */
		tickCount;
		
		const lines: { path: string; key: string }[] = [];
		for (const link of forceLinks) {
			const sourceId = typeof link.source === 'object' ? link.source.id : link.source;
			const targetId = typeof link.target === 'object' ? link.target.id : link.target;
			const source = notes.find(n => n.id === sourceId);
			const target = notes.find(n => n.id === targetId);
			if (source && target) {
				const sw = source.width || 250, sh = source.height || 120;
				const tw = target.width || 250, th = target.height || 120;
				const cx1 = source.x + sw / 2;
				const cy1 = source.y + sh / 2;
				const cx2 = target.x + tw / 2;
				const cy2 = target.y + th / 2;
				const dx = cx2 - cx1, dy = cy2 - cy1;
				
				let pathDefinition;
				if (Math.abs(dx) > Math.abs(dy)) {
					pathDefinition = `M ${cx1} ${cy1} C ${cx1 + dx/2} ${cy1}, ${cx2 - dx/2} ${cy2}, ${cx2} ${cy2}`;
				} else {
					pathDefinition = `M ${cx1} ${cy1} C ${cx1} ${cy1 + dy/2}, ${cx2} ${cy2 - dy/2}, ${cx2} ${cy2}`;
				}
				lines.push({ path: pathDefinition, key: link.id });
			}
		}
		return lines;
	});

	// 수동 연결선 렌더링
	const manualConnectionsPaths = $derived.by(() => {
		/* eslint-disable-next-line @typescript-eslint/no-unused-expressions */
		tickCount;
		
		const result: { id: string; path: string }[] = [];
		for (const conn of manualConns) {
			const source = notes.find(n => n.id === conn.source_node_id);
			const target = notes.find(n => n.id === conn.target_node_id);
			if (source && target) {
				const p1 = getPortCoords(source, conn.source_port);
				const p2 = getPortCoords(target, conn.target_port);
				result.push({
					id: conn.id,
					path: createSCurve(p1.x, p1.y, conn.source_port, p2.x, p2.y, conn.target_port)
				});
			}
		}
		return result;
	});

	const tempConnectionPath = $derived.by(() => {
		/* eslint-disable-next-line @typescript-eslint/no-unused-expressions */
		tickCount;
		if (!isConnecting || !connStartNodeId || !connStartPort) return null;
		const source = notes.find(n => n.id === connStartNodeId);
		if (!source) return null;
		
		const p1 = getPortCoords(source, connStartPort);
		return createSCurve(p1.x, p1.y, connStartPort, connCurrentX, connCurrentY, null);
	});

	function handlePortDown(e: PointerEvent, nodeId: string, port: PortPosition) {
		isConnecting = true;
		connStartNodeId = nodeId;
		connStartPort = port;
		
		const rect = document.querySelector('.main-content')?.getBoundingClientRect();
		if (rect) {
			const cursorX = e.clientX - rect.left;
			const cursorY = e.clientY - rect.top;
			connCurrentX = (cursorX - canvasPanX) / Math.max(0.1, canvasScale);
			connCurrentY = (cursorY - canvasPanY) / Math.max(0.1, canvasScale);
		}
		(e.currentTarget as HTMLElement).setPointerCapture(e.pointerId);
	}

	function handleCanvasPointerMove(e: PointerEvent) {
		if (isConnecting) {
			const rect = document.querySelector('.main-content')?.getBoundingClientRect();
			if (rect) {
				const cursorX = e.clientX - rect.left;
				const cursorY = e.clientY - rect.top;
				connCurrentX = (cursorX - canvasPanX) / Math.max(0.1, canvasScale);
				connCurrentY = (cursorY - canvasPanY) / Math.max(0.1, canvasScale);
			}
		}
	}

	async function handleCanvasPointerUp(e: PointerEvent) {
		if (isConnecting && connStartNodeId) {
			isConnecting = false;

			const targetEl = document.elementFromPoint(e.clientX, e.clientY) as HTMLElement;
			if (targetEl && targetEl.classList.contains('port')) {
				const targetNodeId = targetEl.dataset.nodeid;
				const targetPort = targetEl.dataset.port as PortPosition;
				
				if (targetNodeId && targetNodeId !== connStartNodeId && connStartPort && targetPort) {
					const newConn: ManualConnection = {
						id: genId(),
						canvas_id: activeCanvasId || undefined,
						source_node_id: connStartNodeId,
						source_port: connStartPort,
						target_node_id: targetNodeId,
						target_port: targetPort
					};
					
					const allConns = getLS<ManualConnection[]>(STORAGE_CONNS, []);
					allConns.push(newConn);
					setLS(STORAGE_CONNS, allConns);
					manualConns = [...manualConns, newConn];
				}
			}
			connStartNodeId = null;
			connStartPort = null;
		}
	}

	async function deleteManualConnection(id: string) {
		const allConns = getLS<ManualConnection[]>(STORAGE_CONNS, []);
		setLS(STORAGE_CONNS, allConns.filter(c => c.id !== id));
		manualConns = manualConns.filter(c => c.id !== id);
	}

	function scatterNodes() {
		if (simulation) {
			simulation.alpha(1).restart();
		}
	}
</script>

<svelte:head>
	<title>Nodex</title>
</svelte:head>

<Sidebar 
	canvases={canvases} 
	activeCanvasId={activeCanvasId} 
	onselect={handleSelectCanvas}
	oncreate={handleCreateCanvas}
	ondelete={handleDeleteCanvas}
/>

<!-- 캔버스 영역: Sidebar(250px) 우측에 배치 -->
<div class="main-content" onpointermove={handleCanvasPointerMove} onpointerup={handleCanvasPointerUp}>
	<div class="toolbar">
		{#if activeCanvasId}
			<div class="filter-box">
				<input 
					type="text" 
					placeholder="🔍 노트 필터링 / 동적 중앙노드"
					bind:value={filterKeyword}
					oninput={applyFilterAndRebuild}
				/>
			</div>
			<label class="toggle-link">
				<input type="checkbox" bind:checked={enableAutoLink} />
				[[]] 자동 연결선 표시
			</label>
			<button class="action-btn" onclick={scatterNodes}>물리 시뮬레이션 휘젓기</button>
		{/if}
	</div>

	{#if activeCanvasId}
		<Canvas 
			oncanvasdoubleclick={handleDoubleClick} 
			bind:scale={canvasScale} 
			bind:panX={canvasPanX} 
			bind:panY={canvasPanY}
		>
			<svg 
				class="connections-layer" 
				width="10000" 
				height="10000" 
				style="position: absolute; top: -5000px; left: -5000px; overflow: visible; pointer-events: none;"
			>
				<g transform="translate(5000, 5000)">
					{#if tickCount !== -1} <!-- 강제 업데이트 용도 무옵션 -->
						<!-- 자동 마크다운 연결선 -->
						{#each drawingAutoConnections as line (line.key)}
							<path 
								d={line.path}
								stroke="#4da6ff" 
								stroke-width="3" 
								stroke-dasharray="5,5"
								fill="none"
								opacity="0.5"
								style="pointer-events: none;"
							/>
						{/each}

						<!-- 수동 드래그 연결선 -->
						{#each manualConnectionsPaths as line (line.id)}
							<!-- svelte-ignore a11y_no_static_element_interactions -->
							<!-- svelte-ignore a11y_click_events_have_key_events -->
							<path 
								d={line.path}
								class="manual-path"
								stroke="#ffc107" 
								stroke-width="4" 
								fill="none"
								oncontextmenu={(e) => { e.preventDefault(); deleteManualConnection(line.id); }}
							/>
						{/each}

						<!-- 그림 중인 임시 선 -->
						{#if tempConnectionPath}
							<path 
								d={tempConnectionPath}
								stroke="#ffc107" 
								stroke-width="4" 
								fill="none"
								opacity="0.8"
								style="pointer-events: none;"
							/>
						{/if}
					{/if}
				</g>
			</svg>

			{#if tickCount !== -1}
				{#each notes as note (note.id)}
					<Node 
						id={note.id} 
						x={note.x} 
						y={note.y} 
						pinned={note.pinned}
						text={note.text} 
						entries={note.entries}
						ondragstart={handleDragStart}
						ondrag={handleDrag}
						ondragend={handleDragEnd}
						onpin={handlePinNode}
						onedit={() => handleNodeEdit(note.id, note.text, note.x, note.y)}
						onportdown={handlePortDown}
						onresize={handleNodeResize}
					/>
				{/each}
			{/if}
		</Canvas>

		<BottomInput onsave={handleBottomInputSave} />
	{:else}
		<div class="no-canvas">
			<p>선택된 캔버스가 없습니다. 사이드바에서 새 캔버스를 만들어주세요.</p>
		</div>
	{/if}
</div>

{#if isEditing}
	<NodeEditor 
		x={editTargetX} 
		y={editTargetY} 
		initialText={editInitialText}
		initialEntries={editTargetId ? (notes.find(n => n.id === editTargetId)?.entries || []) : []}
		onsave={handleSaveNote} 
		oncancel={handleCancelEdit} 
	/>
{/if}

<Help />

<style>
	.main-content {
		position: absolute;
		left: 250px; /* Sidebar width */
		top: 0;
		right: 0;
		bottom: 0;
		overflow: hidden;
	}

	.toolbar {
		position: absolute;
		top: 16px;
		right: 16px;
		z-index: 50;
		display: flex;
		gap: 12px;
		align-items: center;
	}

	.filter-box input {
		width: 280px;
		background: rgba(30, 30, 30, 0.9);
		color: #e0e0e0;
		border: 1px solid #3e3e42;
		border-radius: 4px;
		padding: 8px 12px;
		font-size: 13px;
		outline: none;
		box-shadow: 0 4px 12px rgba(0,0,0,0.5);
	}
	.filter-box input:focus {
		border-color: #007acc;
	}

	.toggle-link {
		color: #e0e0e0;
		font-size: 13px;
		display: flex;
		align-items: center;
		gap: 6px;
		cursor: pointer;
	}

	.action-btn {
		background: #333;
		color: #e0e0e0;
		border: 1px solid #555;
		padding: 8px 16px;
		border-radius: 6px;
		cursor: pointer;
		font-size: 13px;
	}

	.action-btn:hover {
		background: #444;
	}

	.manual-path {
		cursor: crosshair;
		pointer-events: stroke;
		transition: stroke 0.2s, stroke-width 0.2s;
	}
	.manual-path:hover {
		stroke: #ff5555;
		stroke-width: 6;
	}

	.no-canvas {
		display: flex;
		align-items: center;
		justify-content: center;
		height: 100%;
		color: #888;
		font-size: 18px;
	}
</style>
