<script lang="ts">
	let {
		canvases,
		activeCanvasId,
		onselect,
		oncreate,
		ondelete
	} = $props<{
		canvases: { id: string; name: string }[];
		activeCanvasId: string | null;
		onselect: (id: string) => void;
		oncreate: (name: string) => void;
		ondelete: (id: string) => void;
	}>();

	let newName = $state('');
	let isCreating = $state(false);

	function handleCreate() {
		if (newName.trim()) {
			oncreate(newName.trim());
			newName = '';
			isCreating = false;
		}
	}

	function handleKeydown(e: KeyboardEvent) {
		if (e.key === 'Enter') handleCreate();
		if (e.key === 'Escape') {
			isCreating = false;
			newName = '';
		}
	}
</script>

<div class="sidebar">
	<div class="header">
		<h2>Nodex 스페이스</h2>
		<button class="add-btn" onclick={() => (isCreating = true)}>+</button>
	</div>

	{#if isCreating}
		<div class="create-form">
			<input
				type="text"
				bind:value={newName}
				onkeydown={handleKeydown}
				placeholder="새 캔버스 이름"
				autofocus
			/>
			<div class="actions">
				<button onclick={handleCreate}>생성</button>
				<button onclick={() => (isCreating = false)}>취소</button>
			</div>
		</div>
	{/if}

	<ul class="canvas-list">
		{#each canvases as canvas}
			<!-- svelte-ignore a11y_click_events_have_key_events -->
			<!-- svelte-ignore a11y_no_noninteractive_element_interactions -->
			<li 
				class={{ active: canvas.id === activeCanvasId }}
				onclick={() => onselect(canvas.id)}
			>
				<span class="name">{canvas.name}</span>
				<button class="delete-btn" onclick={(e) => {
					e.stopPropagation();
					if(confirm('이 캔버스를 삭제할까요? 내부의 모든 노트가 지워집니다.')) {
						ondelete(canvas.id);
					}
				}}>
					&times;
				</button>
			</li>
		{/each}
		{#if canvases.length === 0 && !isCreating}
			<li class="empty">캔버스가 없습니다. +를 눌러 만드세요.</li>
		{/if}
	</ul>
</div>

<style>
	.sidebar {
		width: 250px;
		height: 100vh;
		background-color: #1e1e1e;
		border-right: 1px solid #333;
		display: flex;
		flex-direction: column;
		position: fixed;
		left: 0;
		top: 0;
		z-index: 100;
	}

	.header {
		padding: 16px;
		display: flex;
		justify-content: space-between;
		align-items: center;
		border-bottom: 1px solid #333;
	}

	h2 {
		margin: 0;
		font-size: 16px;
		color: #fff;
		font-weight: 600;
	}

	.add-btn {
		background: #007acc;
		color: white;
		border: none;
		border-radius: 4px;
		width: 24px;
		height: 24px;
		display: flex;
		align-items: center;
		justify-content: center;
		cursor: pointer;
	}

	.add-btn:hover {
		background: #008be6;
	}

	.create-form {
		padding: 12px;
		border-bottom: 1px solid #333;
		background: #252526;
	}

	.create-form input {
		width: 100%;
		padding: 6px;
		background: #3c3c3c;
		border: 1px solid #555;
		color: white;
		border-radius: 4px;
		box-sizing: border-box;
		margin-bottom: 8px;
	}

	.create-form .actions {
		display: flex;
		gap: 8px;
	}

	.create-form button {
		flex: 1;
		padding: 4px;
		background: #444;
		border: none;
		color: white;
		border-radius: 4px;
		cursor: pointer;
	}

	.create-form button:first-child {
		background: #007acc;
	}

	.canvas-list {
		list-style: none;
		margin: 0;
		padding: 0;
		overflow-y: auto;
		flex: 1;
	}

	.canvas-list li {
		padding: 12px 16px;
		cursor: pointer;
		color: #ccc;
		border-bottom: 1px solid #2d2d2d;
		display: flex;
		justify-content: space-between;
		align-items: center;
		transition: background-color 0.1s;
	}

	.canvas-list li:hover {
		background-color: #2a2d2e;
	}

	.canvas-list li.active {
		background-color: #37373d;
		color: #fff;
		font-weight: bold;
		border-left: 3px solid #007acc;
	}

	.delete-btn {
		background: transparent;
		color: #888;
		border: none;
		cursor: pointer;
		font-size: 16px;
		padding: 0 4px;
		display: none;
	}

	.canvas-list li:hover .delete-btn {
		display: block;
	}

	.delete-btn:hover {
		color: #ff5555;
	}

	.empty {
		color: #666 !important;
		cursor: default !important;
		text-align: center;
		padding: 24px 16px !important;
	}
	.empty:hover {
		background: none !important;
	}
</style>
