<script lang="ts">
	let { onsave } = $props<{
		onsave: (text: string) => void;
	}>();

	let text = $state('');

	function handleKeydown(e: KeyboardEvent) {
		// Shift + Enter 또는 Enter 단독이 줄바꿈이 될 수도 있지만,
		// 요구사항: Enter는 줄바꿈, Ctrl+Enter는 저장
		if (e.key === 'Enter' && e.ctrlKey) {
			e.preventDefault();
			handleSave();
		}
	}

	function handleSave() {
		if (text.trim()) {
			onsave(text.trim());
			text = '';
		}
	}
</script>

<div class="bottom-input">
	<div class="input-container">
		<textarea
			bind:value={text}
			onkeydown={handleKeydown}
			placeholder="여기에 노트를 입력하세요... (Enter: 줄바꿈, Ctrl+Enter: 캔버스 중앙에 새 노트 저장)"
		></textarea>
		<button class="save-btn" onclick={handleSave} disabled={!text.trim()}>
			기록
		</button>
	</div>
</div>

<style>
	.bottom-input {
		position: fixed;
		bottom: 24px;
		left: 50%;
		transform: translateX(-50%);
		width: 600px;
		max-width: 90vw;
		background: rgba(30, 30, 30, 0.85);
		backdrop-filter: blur(10px);
		border-radius: 12px;
		border: 1px solid #444;
		box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
		z-index: 100;
		padding: 12px;
	}

	.input-container {
		display: flex;
		gap: 12px;
		align-items: flex-end;
	}

	textarea {
		flex: 1;
		background: transparent;
		border: none;
		color: #e0e0e0;
		font-size: 15px;
		resize: none;
		height: 48px;
		min-height: 48px;
		max-height: 200px;
		outline: none;
		font-family: inherit;
		line-height: 1.5;
		transition: height 0.2s;
	}

	textarea:focus {
		height: 80px;
	}

	.save-btn {
		background: #007acc;
		color: white;
		border: none;
		border-radius: 8px;
		padding: 12px 24px;
		font-weight: bold;
		cursor: pointer;
		height: 44px;
		transition: background 0.2s;
	}

	.save-btn:hover {
		background: #008be6;
	}

	.save-btn:disabled {
		background: #444;
		color: #888;
		cursor: not-allowed;
	}
</style>
