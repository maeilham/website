<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import * as XtermPkg from '@xterm/xterm';
	import * as FitPkg from '@xterm/addon-fit';
	import '@xterm/xterm/css/xterm.css';

	const Terminal = (XtermPkg as any).Terminal ?? (XtermPkg as any).default?.Terminal;
	const FitAddon  = (FitPkg as any).FitAddon  ?? (FitPkg as any).default?.FitAddon;

	let { action = '', token = '' }: { action?: string; token?: string } = $props();

	const API_WS = (import.meta.env.VITE_API_URL ?? 'http://localhost:8080')
		.replace(/^http/, 'ws');

	let termEl: HTMLDivElement;
	let term: any;
	let fitAddon: any;
	let ws: WebSocket | null = null;

	onMount(() => {
		term = new Terminal({
			fontFamily: '"DM Mono", "Fira Code", "Cascadia Code", monospace',
			fontSize: 12,
			lineHeight: 1.5,
			theme: {
				background:   '#1e1e1e',
				foreground:   '#d4d4d4',
				cursor:       '#ffffff',
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
		return () => ro.disconnect();
	});

	onDestroy(() => {
		ws?.close();
		term?.dispose();
	});

	function connectWS() {
		const params = new URLSearchParams();
		if (action) params.set('action', action);
		if (token) params.set('token', token);
		const query = params.toString() ? `?${params}` : '';
		ws = new WebSocket(`${API_WS}/ws/terminal${query}`);
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
</script>

<div class="home-terminal">
	<div class="term-wrap" bind:this={termEl}></div>
</div>

<style>
	.home-terminal {
		width: 100%;
	}

	.term-wrap {
		overflow: hidden;
		background: #1e1e1e;
		aspect-ratio: 4 / 3;
		padding: 4px;
	}

	:global(.xterm-viewport::-webkit-scrollbar) { width: 6px; }
	:global(.xterm-viewport::-webkit-scrollbar-thumb) {
		background: #333;
		border-radius: 3px;
	}
</style>
