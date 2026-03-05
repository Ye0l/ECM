<script lang="ts">
	export type AccountEntry = {
		id: string; // 폼 핸들링용 고유 ID
		accountName: string;
		amount: number;
		side: 'left' | 'right'; 
	};

	let { x, y, initialText = '', initialEntries = [], onsave, oncancel } = $props<{
		x: number;
		y: number;
		initialText?: string;
		initialEntries?: AccountEntry[];
		onsave: (text: string, x: number, y: number, entries: AccountEntry[]) => void;
		oncancel: () => void;
	}>();

	// svelte-ignore state_referenced_locally
	let text = $state(initialText);
	// svelte-ignore state_referenced_locally
	// 클론을 만들고 id가 없다면 부여. amounts는 string으로 다루기 편하도록 래핑
	// svelte-ignore state_referenced_locally
	let entries = $state(initialEntries.map((e: AccountEntry) => ({
		...e,
		id: e.id || Math.random().toString(36).substr(2, 9),
		amountStr: e.amount ? e.amount.toString() : ''
	})));
	
	let textarea: HTMLTextAreaElement | undefined = $state();

	$effect(() => {
		if (textarea) textarea.focus();
	});

	function handleKeydown(e: KeyboardEvent) {
		if (e.key === 'Enter' && e.ctrlKey) {
			handleSave();
		} else if (e.key === 'Escape') {
			oncancel();
		}
	}

	function handleAddEntry(side: 'left' | 'right') {
		entries = [...entries, {
			id: Math.random().toString(36).substr(2, 9),
			accountName: '',
			amount: 0,
			amountStr: '',
			side
		}];
	}

	function handleRemoveEntry(id: string) {
		entries = entries.filter((e: AccountEntry & {amountStr: string}) => e.id !== id);
	}

	function handleSave() {
		if (text.trim() || entries.length > 0) {
			// 입력 값 정제 (숫자만 추출, 빈 계정명은 무시)
			const cleanEntries: AccountEntry[] = entries
				.filter((e: AccountEntry & {amountStr: string}) => e.accountName.trim() !== '')
				.map((e: AccountEntry & {amountStr: string}) => {
					const parsedAmount = parseInt(e.amountStr.replace(/[^0-9-]/g, ''), 10);
					return {
						id: e.id,
						accountName: e.accountName.trim(),
						amount: isNaN(parsedAmount) ? 0 : parsedAmount,
						side: e.side
					};
				});

			onsave(text.trim(), x, y, cleanEntries);
		} else {
			oncancel();
		}
	}
	
	const leftEntries = $derived(entries.filter((e: AccountEntry & {amountStr: string}) => e.side === 'left'));
	const rightEntries = $derived(entries.filter((e: AccountEntry & {amountStr: string}) => e.side === 'right'));
</script>

<!-- svelte-ignore a11y_click_events_have_key_events -->
<!-- svelte-ignore a11y_no_static_element_interactions -->
<div class="editor-overlay" onclick={oncancel}>
	<div class="editor-modal" onclick={(e) => e.stopPropagation()}>
		<h3>노드 및 거래 내역 편집</h3>
		
		<textarea
			bind:this={textarea}
			bind:value={text}
			onkeydown={handleKeydown}
			placeholder="내용 또는 메모를 입력하세요... [[자동연결태그]] (Ctrl+Enter 저장, ESC 취소)"
		></textarea>
		
		<div class="double-entry-grid">
			<!-- 차변 (왼쪽) -->
			<div class="side-panel left-side">
				<div class="side-header">
					<h4>차변 (왼쪽)</h4>
					<button class="add-btn" onclick={() => handleAddEntry('left')}>+ 항목 추가</button>
				</div>
				<div class="entry-list">
					{#each leftEntries as entry (entry.id)}
						<div class="entry-row">
							<input type="text" placeholder="계정명" bind:value={entry.accountName} class="acc-input" />
							<input type="text" placeholder="금액" bind:value={entry.amountStr} class="amt-input" />
							<button class="del-btn" onclick={() => handleRemoveEntry(entry.id)}>×</button>
						</div>
					{/each}
					{#if leftEntries.length === 0}
						<div class="empty-state">항목이 없습니다.</div>
					{/if}
				</div>
			</div>

			<!-- 대변 (오른쪽) -->
			<div class="side-panel right-side">
				<div class="side-header">
					<h4>대변 (오른쪽)</h4>
					<button class="add-btn" onclick={() => handleAddEntry('right')}>+ 항목 추가</button>
				</div>
				<div class="entry-list">
					{#each rightEntries as entry (entry.id)}
						<div class="entry-row">
							<input type="text" placeholder="계정명" bind:value={entry.accountName} class="acc-input" />
							<input type="text" placeholder="금액" bind:value={entry.amountStr} class="amt-input" />
							<button class="del-btn" onclick={() => handleRemoveEntry(entry.id)}>×</button>
						</div>
					{/each}
					{#if rightEntries.length === 0}
						<div class="empty-state">항목이 없습니다.</div>
					{/if}
				</div>
			</div>
		</div>

		<div class="actions">
			<button class="cancel" onclick={oncancel}>취소</button>
			<button class="save" onclick={handleSave}>저장</button>
		</div>
	</div>
</div>

<style>
	.editor-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.5);
		backdrop-filter: blur(2px);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 1000;
	}
	.editor-modal {
		background: #252526;
		border: 1px solid #3e3e42;
		border-radius: 8px;
		padding: 20px;
		width: 500px; /* 기존보다 너비 확대 */
		max-width: 90vw;
		box-shadow: 0 10px 30px rgba(0, 0, 0, 0.6);
		display: flex;
		flex-direction: column;
		gap: 16px;
	}
	h3 {
		margin: 0;
		font-size: 16px;
		color: #ffffff;
		font-weight: 600;
	}
	
	textarea {
		width: 100%;
		height: 80px;
		background: #1e1e1e;
		color: #cccccc;
		border: 1px solid #3e3e42;
		border-radius: 4px;
		padding: 12px;
		font-size: 14px;
		resize: vertical;
		max-height: 40vh;
		overflow-y: auto;
		font-family: inherit;
		outline: none;
	}
	textarea:focus {
		border-color: #007acc;
	}

	.double-entry-grid {
		display: grid;
		grid-template-columns: 1fr 1fr;
		gap: 16px;
		border-top: 1px solid #3e3e42;
		padding-top: 16px;
	}

	.side-header {
		display: flex;
		justify-content: space-between;
		align-items: center;
		margin-bottom: 8px;
	}
	.side-header h4 {
		margin: 0;
		font-size: 13px;
		color: #a0a0a0;
	}

	.add-btn {
		background: transparent;
		color: #4da6ff;
		padding: 4px 8px;
		font-size: 12px;
	}
	.add-btn:hover {
		background: rgba(77, 166, 255, 0.1);
	}

	.entry-list {
		display: flex;
		flex-direction: column;
		gap: 8px;
	}

	.empty-state {
		color: #666;
		font-size: 12px;
		text-align: center;
		padding: 12px 0;
		background: rgba(0, 0, 0, 0.2);
		border-radius: 4px;
	}

	.entry-row {
		display: flex;
		gap: 4px;
		align-items: center;
	}

	input[type="text"] {
		background: #1e1e1e;
		color: #cccccc;
		border: 1px solid #3e3e42;
		border-radius: 4px;
		padding: 6px 8px;
		font-size: 13px;
		outline: none;
	}
	input[type="text"]:focus {
		border-color: #007acc;
	}

	.acc-input {
		flex: 1.5;
		min-width: 0;
	}
	.amt-input {
		flex: 1;
		min-width: 0;
	}

	.del-btn {
		background: transparent;
		color: #ff5555;
		font-size: 16px;
		padding: 0 4px;
		line-height: 1;
	}
	.del-btn:hover {
		background: rgba(255, 85, 85, 0.1);
	}

	.actions {
		display: flex;
		justify-content: flex-end;
		gap: 8px;
		margin-top: 12px;
	}
	button {
		padding: 8px 16px;
		border: none;
		border-radius: 4px;
		cursor: pointer;
		font-size: 14px;
		font-weight: 500;
	}
	.cancel {
		background: transparent;
		color: #a0a0a0;
	}
	.cancel:hover {
		background: rgba(255, 255, 255, 0.1);
		color: #e0e0e0;
	}
	.save {
		background: #007acc;
		color: white;
	}
	.save:hover {
		background: #008be6;
	}
</style>
