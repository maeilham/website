<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import * as XtermPkg from '@xterm/xterm';
	import * as FitPkg from '@xterm/addon-fit';
	import '@xterm/xterm/css/xterm.css';

	const Terminal = (XtermPkg as any).Terminal ?? (XtermPkg as any).default?.Terminal;
	const FitAddon  = (FitPkg as any).FitAddon  ?? (FitPkg as any).default?.FitAddon;

	let { onClose }: { onClose: () => void } = $props();

	const API_WS = (import.meta.env.VITE_API_URL ?? 'http://localhost:8080')
		.replace(/^http/, 'ws');

	let containerEl: HTMLDivElement;
	let termEl: HTMLDivElement;
	let term: any;
	let fitAddon: any;
	let ws: WebSocket | null = null;
	let mode = $state<'floating' | 'docked'>('floating');

	// ── 초기화 ────────────────────────────────────────────────────────────────
	onMount(() => {
		// 초기 위치: bottom-right
		const top  = window.innerHeight - 482 - 12;
		const left = window.innerWidth  - 705 - 12;
		containerEl.style.top    = `${top}px`;
		containerEl.style.left   = `${left}px`;
		containerEl.style.right  = '12px';
		containerEl.style.bottom = '12px';

		// scale-in 애니메이션
		containerEl.style.transform = 'scale(0)';
		requestAnimationFrame(() => {
			containerEl.style.transform = 'scale(1)';
		});

		term = new Terminal({
			fontFamily: '"DM Mono", "Fira Code", "Cascadia Code", monospace',
			fontSize: 14,
			lineHeight: 1.5,
			theme: {
				background:   '#1e1e1e',
				foreground:   '#d4d4d4',
				cursor:       '#90ee90',
				cursorAccent: '#1e1e1e',
			},
			cursorBlink: true,
			scrollback: 500,
		});

		fitAddon = new FitAddon();
		term.loadAddon(fitAddon);
		term.open(termEl);
		fitAddon.fit();

		connectWS();

		const ro = new ResizeObserver(() => fitAddon?.fit());
		ro.observe(termEl);

		makeDraggable();

		return () => ro.disconnect();
	});

	onDestroy(() => {
		ws?.close();
		term?.dispose();
	});

	// ── WebSocket ─────────────────────────────────────────────────────────────
	function connectWS() {
		ws = new WebSocket(`${API_WS}/ws/terminal`);
		ws.binaryType = 'arraybuffer';
		ws.onmessage = (e) => {
			if (e.data instanceof ArrayBuffer) term.write(new Uint8Array(e.data));
			else term.write(e.data);
		};
		ws.onclose = () => term.write('\r\n\x1b[2m연결이 종료됐습니다.\x1b[0m\r\n');
		ws.onerror = () => term.write('\r\n\x1b[31m서버에 연결할 수 없습니다.\x1b[0m\r\n');
		term.onData((data: string) => {
			if (ws?.readyState === WebSocket.OPEN) ws.send(data);
		});
	}

	// ── 독/플로팅 ─────────────────────────────────────────────────────────────
	function toggleDock() {
		containerEl.style.removeProperty('left');
		containerEl.style.removeProperty('top');

		if (mode === 'floating') {
			containerEl.style.width  = '100vw';
			containerEl.style.height = '300px';
			containerEl.style.right  = '0px';
			containerEl.style.bottom = '0px';
			containerEl.setAttribute('data-mode', 'docked');
			mode = 'docked';
		} else {
			containerEl.style.width  = '705px';
			containerEl.style.height = '482px';
			containerEl.style.right  = '12px';
			containerEl.style.bottom = '12px';
			containerEl.setAttribute('data-mode', 'floating');
			mode = 'floating';
		}
		setTimeout(() => fitAddon?.fit(), 350);
	}

	// ── 닫기 ─────────────────────────────────────────────────────────────────
	function hide() {
		containerEl.style.transform = 'scale(0)';
		setTimeout(onClose, 300);
	}

	// ── 드래그 ────────────────────────────────────────────────────────────────
	function makeDraggable() {
		const header = containerEl.querySelector('.titlebar') as HTMLElement;
		let isDown = false;
		let startX = 0, startY = 0, offsetX = 0, offsetY = 0;

		header.addEventListener('mousedown', (e) => {
			if ((e.target as HTMLElement).closest('.icon-btn')) return;
			if (mode === 'docked') return;
			isDown = true;
			startX  = e.clientX;
			startY  = e.clientY;
			offsetX = containerEl.offsetLeft;
			offsetY = containerEl.offsetTop;
			document.addEventListener('mousemove', onMove);
			document.addEventListener('mouseup', onUp);
		});

		const onMove = (e: MouseEvent) => {
			if (!isDown) return;
			e.preventDefault();
			containerEl.style.left   = `${offsetX + (e.clientX - startX)}px`;
			containerEl.style.top    = `${offsetY + (e.clientY - startY)}px`;
			containerEl.style.right  = 'auto';
			containerEl.style.bottom = 'auto';
		};

		const onUp = () => {
			isDown = false;
			document.removeEventListener('mousemove', onMove);
			document.removeEventListener('mouseup', onUp);
		};
	}

	function handleKeydown(e: KeyboardEvent) {
		if (e.key === 'Escape') hide();
	}
</script>

<svelte:window onkeydown={handleKeydown} />

<div class="popup" bind:this={containerEl} data-mode="floating">
	<div class="titlebar">
		<button class="icon-btn" onclick={toggleDock} aria-label="dock/float">
			{#if mode === 'docked'}
				<svg width="14" height="14" viewBox="0 0 9 9" fill="none" xmlns="http://www.w3.org/2000/svg">
					<path fill-rule="evenodd" clip-rule="evenodd" d="M1 0H0V1H1V0ZM3 0H2V1H3V0ZM4 0H9V5H4V0ZM1 2H0V3H1V2ZM0 4H1V5H0V4ZM1 6H0V7H1V6ZM0 8H1V9H0V8ZM3 8H2V9H3V8ZM4 8H5V9H4V8ZM7 8H6V9H7V8ZM8 8H9V9H8V8ZM9 6H8V7H9V6Z" fill="currentColor"/>
				</svg>
			{:else}
				<svg width="14" height="14" viewBox="0 0 9 9" fill="none" xmlns="http://www.w3.org/2000/svg">
					<path fill-rule="evenodd" clip-rule="evenodd" d="M0 0H1V1H0V0ZM1 2H0V3H1V2ZM1 5H8V8H1V5ZM0 4H1H8H9V5V8V9H8H1H0V8V5V4ZM3 0H2V1H3V0ZM4 0H5V1H4V0ZM7 0H6V1H7V0ZM8 0H9V1H8V0ZM9 2H8V3H9V2Z" fill="currentColor"/>
				</svg>
			{/if}
		</button>
		<span class="title">MAEILHAM</span>
		<button class="icon-btn" onclick={hide} aria-label="닫기">
			<svg width="14" height="14" viewBox="0 0 9 9" fill="none" xmlns="http://www.w3.org/2000/svg">
				<path fill-rule="evenodd" clip-rule="evenodd" d="M0 0H1V1H0V0ZM2 2H1V1H2V2ZM3 3H2V2H3V3ZM4 4H3V3H4V4ZM5 4H4V5H3V6H2V7H1V8H0V9H1V8H2V7H3V6H4V5H5V6H6V7H7V8H8V9H9V8H8V7H7V6H6V5H5V4ZM6 3V4H5V3H6ZM7 2V3H6V2H7ZM8 1V2H7V1H8ZM8 1V0H9V1H8Z" fill="currentColor"/>
			</svg>
		</button>
	</div>
	<div class="term-wrap" bind:this={termEl}></div>
</div>

<style>
	.popup {
		position: fixed;
		width: 705px;
		height: 482px;
		border: 1px solid #011627;
		z-index: 9999;
		display: flex;
		flex-direction: column;
		padding: 8px 4px 4px;
		gap: 8px;
		background: rgba(255, 255, 255, 0.15);
		backdrop-filter: blur(12px);
		-webkit-backdrop-filter: blur(12px);
		box-shadow: 0 4px 30px rgba(0, 0, 0, 0.15);
		transform-origin: bottom right;
		transition: transform 0.25s cubic-bezier(0.34, 1.2, 0.64, 1),
		            width   0.3s  cubic-bezier(0.4, 0, 0.2, 1),
		            height  0.3s  cubic-bezier(0.4, 0, 0.2, 1),
		            right   0.3s  cubic-bezier(0.4, 0, 0.2, 1),
		            bottom  0.3s  cubic-bezier(0.4, 0, 0.2, 1);
	}

	.titlebar {
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 1px 4px;
		flex-shrink: 0;
		user-select: none;
		cursor: grab;
		line-height: 1;
	}
	.titlebar:active { cursor: grabbing; }

	.title {
		font-size: 13px;
		font-weight: 600;
		font-family: 'DM Mono', 'Fira Code', monospace;
		text-transform: uppercase;
		letter-spacing: 0.08em;
		color: #011627;
	}

	.icon-btn {
		background: none;
		border: none;
		cursor: pointer;
		padding: 2px 4px;
		color: #011627;
		opacity: 0.45;
		display: flex;
		align-items: center;
		transition: opacity 0.15s;
	}
	.icon-btn:hover { opacity: 1; }

	.term-wrap {
		flex: 1;
		overflow: hidden;
		background: #1e1e1e;
		border-radius: 4px;
		border: 1px solid transparent;
	}

	:global(.xterm-viewport::-webkit-scrollbar) { width: 6px; }
	:global(.xterm-viewport::-webkit-scrollbar-thumb) {
		background: #333;
		border-radius: 3px;
	}
</style>
