<script lang="ts">
	let isOpen = $state(false);

	function toggle() {
		isOpen = !isOpen;
	}

	$effect(() => {
		const handleKeydown = (e: KeyboardEvent) => {
			if (e.key === 'Escape' && isOpen) {
				isOpen = false;
			}
			// 예시: 슬래시(/) 입력 등 커맨드 창을 띄울 수 있는 훅을 여기에 추가할 수 있습니다.
		};
		window.addEventListener('keydown', handleKeydown);
		return () => window.removeEventListener('keydown', handleKeydown);
	});
</script>

<button class="help-btn" onclick={toggle} aria-label="도움말">
	?
</button>

{#if isOpen}
	<!-- svelte-ignore a11y_click_events_have_key_events -->
	<!-- svelte-ignore a11y_no_static_element_interactions -->
	<div class="help-overlay" onclick={toggle}>
		<div class="help-modal" onclick={(e) => e.stopPropagation()}>
			<h2>Nodex 릴리즈 및 도움말</h2>
			<div class="content">
				<h3>기본 조작 방법</h3>
				<ul>
					<li><strong>새 노드 작성:</strong> 캔버스 빈 공간을 더블 클릭하면 마크다운 텍스트 편집 모드가 열립니다.</li>
					<li><strong>포트(수동) 선 연결:</strong> 각 노드 상하좌우 화살표 포인트를 마우스로 잡고 당겨(드래그) 다른 노드로 놓으면 영구적인 노란 실선이 연결됩니다. 지우려면 연결된 선 위에 마우스 우클릭을 하세요.</li>
					<li><strong>자동 마크다운 선 연결:</strong> 상단 <code>[[]] 자동 연결</code> 체크박스를 켜두고 노트 텍스트 내에 <code>[[다른노트이름]]</code>을 적으면 자동으로 점선 파란 선이 이어집니다.</li>
					<li><strong>임시 저장:</strong> 작성 중인 모달이나 마크다운 창에서 <kbd>Ctrl</kbd> + <kbd>Enter</kbd>를 누르면 즉시 저장됩니다.</li>
					<li><strong>수정:</strong> 아무 텍스트 노드나 더블클릭하면 다시 수정할 수 있습니다.</li>
					<li><strong>마크다운 및 노드 연결:</strong> 마크다운 문법이 자동 렌더링되며 텍스트 내에 <code>[[대상 텍스트]]</code> 형태의 텍스트를 포함하면 대상 텍스트가 있는 노드와 연결선이 생성됩니다.</li>
					<li><strong>스페이스(캔버스) 관리:</strong> 좌측 사이드바 패널에서 별도의 스페이스를 생성하고 삭제할 수 있습니다.</li>
				</ul>

				<h3>업데이트 내역 (도움말 커맨드 대체)</h3>
				<div class="update-log">
					<p><strong>[v2.0 적용된 수정 및 변경 사항]</strong></p>
					<ul>
						<li>PostgreSQL 데이터베이스 연동으로 완벽한 상태 저장 구현</li>
						<li>좌측 멀티 캔버스(스페이스) 전환 사이드바 추가</li>
						<li>하단 상시 텍스트 입력 패널 추가 (Ctrl+Enter 저장)</li>
						<li>NodeEditor 텍스트에어리어 오버플로우 문제 해결</li>
						<li>개별 노드 마크다운(Markdown) 자동 렌더링 도입 (+더블클릭 인라인 수정)</li>
						<li><code>[[ ]]</code> 정규식 매칭을 이용한 노드 간 시각적 SVG 연결선 구현</li>
						<li>화면 우측 상단 자동 정렬 (그리드 레이아웃) 버튼 추가 기능 구현</li>
					</ul>
				</div>
			</div>
			<div class="footer">
				<button class="close-btn" onclick={toggle}>닫기</button>
			</div>
		</div>
	</div>
{/if}

<style>
	.help-btn {
		position: fixed;
		bottom: 24px;
		right: 24px;
		width: 48px;
		height: 48px;
		border-radius: 50%;
		background-color: #007acc;
		color: white;
		font-size: 24px;
		font-weight: bold;
		border: none;
		cursor: pointer;
		box-shadow: 0 4px 12px rgba(0, 0, 0, 0.4);
		z-index: 900;
		transition: transform 0.2s, background-color 0.2s;
	}
	.help-btn:hover {
		background-color: #008be6;
		transform: scale(1.05);
	}
	.help-overlay {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		background: rgba(0, 0, 0, 0.6);
		backdrop-filter: blur(3px);
		display: flex;
		align-items: center;
		justify-content: center;
		z-index: 1000;
	}
	.help-modal {
		background: #252526;
		border: 1px solid #3e3e42;
		border-radius: 12px;
		padding: 24px;
		width: 500px;
		max-width: 90vw;
		box-shadow: 0 10px 40px rgba(0, 0, 0, 0.6);
		display: flex;
		flex-direction: column;
		gap: 20px;
	}
	h2 {
		margin: 0;
		font-size: 20px;
		color: #ffffff;
		border-bottom: 1px solid #444;
		padding-bottom: 12px;
	}
	h3 {
		margin: 16px 0 8px 0;
		font-size: 16px;
		color: #007acc;
	}
	.content {
		color: #d4d4d4;
		font-size: 14px;
		line-height: 1.6;
	}
	ul {
		margin: 0;
		padding-left: 20px;
	}
	li {
		margin-bottom: 8px;
	}
	kbd {
		background: #333;
		border: 1px solid #555;
		border-radius: 4px;
		padding: 2px 6px;
		font-size: 12px;
		color: #eee;
	}
	.update-log {
		background: #1e1e1e;
		border: 1px solid #333;
		padding: 12px;
		border-radius: 6px;
		margin-top: 10px;
	}
	.update-log p {
		margin-top: 0;
	}
	.footer {
		display: flex;
		justify-content: flex-end;
		border-top: 1px solid #444;
		padding-top: 16px;
	}
	.close-btn {
		background: #444;
		color: white;
		border: none;
		border-radius: 6px;
		padding: 8px 24px;
		font-size: 14px;
		cursor: pointer;
	}
	.close-btn:hover {
		background: #555;
	}
</style>
