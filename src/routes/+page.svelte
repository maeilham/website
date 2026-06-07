<script lang="ts">
	import { page } from "$app/stores";

	const API = import.meta.env.VITE_API_URL ?? "";

	// ── URL 상태 ──────────────────────────────────────────────────────────────
	const urlStatus = $derived($page.url.searchParams.get("status"));
	const urlAction = $derived($page.url.searchParams.get("action"));
	const urlToken  = $derived($page.url.searchParams.get("token"));
	const showUnsubscribeModal = $derived(urlAction === "unsubscribe" && !!urlToken);

	// status 파라미터 읽은 뒤 URL 클린업 (새로고침하면 초기 화면)
	$effect(() => {
		if (urlStatus && typeof window !== "undefined") {
			history.replaceState({}, "", "/");
		}
	});

	// ── 전광판 ────────────────────────────────────────────────────────────────
	type TickerItem = { text: string; sep?: boolean };
	const defaultTicker: TickerItem[] = [
		{ text: "매일 아침, 질문 하나가 도착합니다" },
		{ text: "——", sep: true },
		{ text: "아는 것 같았는데 막히는 것들, 함께 생각해봐요" },
		{ text: "——", sep: true },
		{ text: "DAILY QUESTIONS · GITHUB DISCUSSION · MAEILHAM" },
		{ text: "——", sep: true },
		{ text: "백엔드 · 프론트엔드 · CS 기초" },
		{ text: "——", sep: true },
	];
	const statusTicker: Record<string, TickerItem[]> = {
		confirmed: [
			{ text: "구독이 완료됐습니다" },
			{ text: "——", sep: true },
			{ text: "내일부터 매일 아침 질문이 도착합니다" },
			{ text: "——", sep: true },
			{ text: "함께 답을 만들어가요" },
			{ text: "——", sep: true },
		],
		unsubscribed: [
			{ text: "구독이 해지됐습니다" },
			{ text: "——", sep: true },
			{ text: "그동안 함께해주셔서 감사합니다" },
			{ text: "——", sep: true },
			{ text: "언제든 다시 구독할 수 있습니다" },
			{ text: "——", sep: true },
		],
		invalid: [
			{ text: "링크가 유효하지 않습니다" },
			{ text: "——", sep: true },
			{ text: "만료됐거나 이미 사용된 링크입니다" },
			{ text: "——", sep: true },
			{ text: "다시 구독을 시도해주세요" },
			{ text: "——", sep: true },
		],
	};
	const tickerItems = $derived(urlStatus && statusTicker[urlStatus] ? statusTicker[urlStatus] : defaultTicker);
	const tickerHighlight = $derived(urlStatus === "confirmed");
	const tickerWarning   = $derived(urlStatus === "invalid");
	const tickerStatic    = $derived(urlStatus === "confirmed");

	// ── 샘플 질문 ─────────────────────────────────────────────────────────────
	const letters = [
		{ tag: "백엔드",     date: "2026년 5월 30일", question: "인덱스(Index)란 무엇이고, 언제 사용하면 좋을까요?",  preview: "DB 인덱스는 데이터 검색 속도를 높이기 위해 별도로 관리하는 자료구조입니다. B-Tree 구조로 O(log n) 탐색을 지원합니다." },
		{ tag: "프론트엔드", date: "2026년 5월 29일", question: "브라우저의 렌더링 과정을 설명해주세요.",              preview: "HTML을 파싱해 DOM을, CSS를 파싱해 CSSOM을 생성하고, 렌더 트리를 만든 후 레이아웃과 페인팅을 수행합니다." },
		{ tag: "공통",       date: "2026년 5월 28일", question: "RESTful API 설계 원칙에 대해 설명해주세요.",          preview: "REST는 자원(Resource), 행위(HTTP Method), 표현(Representation)으로 구성됩니다. 무상태성과 일관된 인터페이스가 핵심입니다." },
		{ tag: "백엔드",     date: "2026년 5월 27일", question: "트랜잭션의 ACID 속성에 대해 설명해주세요.",           preview: "원자성(Atomicity), 일관성(Consistency), 격리성(Isolation), 지속성(Durability)으로 데이터베이스 안정성을 보장합니다." },
	];
	let letterIdx = $state(0);
	const currentLetter = $derived(letters[letterIdx]);

	// ── 구독 ─────────────────────────────────────────────────────────────────
	type Phase = "idle" | "loading" | "sent" | "error";
	let phase    = $state<Phase>("idle");
	let email    = $state("");
	let errorMsg = $state("");

	async function subscribe() {
		if (!email) { errorMsg = "이메일을 입력해주세요."; return; }
		phase = "loading";
		errorMsg = "";
		const res = await fetch(`${API}/api/subscribe`, {
			method: "POST",
			headers: { "Content-Type": "application/json" },
			body: JSON.stringify({ email }),
		}).catch(() => null);
		if (!res) { errorMsg = "네트워크 오류가 발생했습니다."; phase = "error"; return; }
		if (res.ok) {
			phase = "sent";
		} else {
			const data = await res.json().catch(() => ({}));
			errorMsg = data.error ?? "오류가 발생했습니다.";
			phase = "error";
		}
	}

	// ── 구독 해지 모달 ────────────────────────────────────────────────────────
	let unsubscribeLoading = $state(false);

	async function confirmUnsubscribe() {
		if (!urlToken) return;
		unsubscribeLoading = true;
		const res = await fetch(`${API}/api/unsubscribe?token=${urlToken}`, { method: "POST" }).catch(() => null);
		window.location.href = res?.ok ? "/?status=unsubscribed" : "/?status=invalid";
	}
</script>

<svelte:head>
	<title>매일함</title>
</svelte:head>

<svelte:window onkeydown={(e) => { if (e.key === "Enter" && phase === "idle") subscribe(); }} />

<div class="page">

	<!-- 로고 -->
	<header class="header">
		<span class="logo">매일함</span>
	</header>

	<!-- 전광판 -->
	<div class="ticker-wrap" class:ticker-highlight={tickerHighlight} class:ticker-warning={tickerWarning} class:ticker-static={tickerStatic}>
		<div class="ticker-track">
			{#each tickerStatic ? [1] : Array(2) as _}
				{#each tickerItems as item}
					{#if item.sep}
						{#if !tickerStatic}<span class="ticker-sep">{item.text}</span>{/if}
					{:else}
						<span class="ticker-item">{item.text}</span>
					{/if}
				{/each}
			{/each}
		</div>
	</div>

	<!-- 메인 콘텐츠 -->
	<main class="content">

		{#if showUnsubscribeModal}
			<!-- 구독 해지 확인 -->
			<div class="intro">
				<p class="unsub-label">구독 해지</p>
				<h1 class="intro-title">정말 해지하시겠어요?</h1>
				<p class="intro-desc">해지 후에도 언제든 다시 구독할 수 있습니다.</p>
			</div>
			<div class="subscribe-form">
				<div class="input-row">
					<button class="subscribe-btn" onclick={confirmUnsubscribe} disabled={unsubscribeLoading}>
						{unsubscribeLoading ? "처리 중..." : "네, 해지할게요"}
					</button>
					<a href="/" class="cancel-link">취소</a>
				</div>
			</div>
		{:else}
			<!-- 서비스 설명 -->
			<div class="intro">
				<h1 class="intro-title">매일 아침,<br>질문 하나가 도착합니다.</h1>
				<p class="intro-desc">
					아는 것 같았는데 막히는 것들 — 백엔드, 프론트엔드, CS 기초를
					매일 한 통씩 받아보세요. GitHub Discussion에서 함께 답을 만들어가요.
				</p>
			</div>

			<!-- 구독 폼 -->
			<div class="subscribe-form">
				{#if phase === "sent"}
					<p class="sent-msg">확인 메일을 보냈습니다 — 메일함을 확인해주세요.</p>
				{:else}
					<div class="input-row">
						<input
							type="email"
							bind:value={email}
							placeholder="이메일 주소를 입력하세요"
							class="email-input"
							disabled={phase === "loading"}
							onkeydown={(e) => e.key === "Enter" && subscribe()}
						/>
						<button class="subscribe-btn" onclick={subscribe} disabled={phase === "loading"}>
							{phase === "loading" ? "..." : "구독하기 →"}
						</button>
					</div>
					{#if errorMsg}
						<p class="error-msg">{errorMsg}</p>
					{/if}
				{/if}
			</div>
		{/if}

	</main>

	<!-- 푸터 -->
	<footer class="footer">
		<span>매일함으로, 매일 함.</span>
	</footer>

</div>


<style>
	:global(body) {
		background: #fff;
	}

	/* ── 레이아웃 ── */
	.page {
		min-height: 100vh;
		display: flex;
		flex-direction: column;
		max-width: 600px;
		margin: 0 auto;
		padding: 0 24px;
	}

	/* ── 헤더 ── */
	.header {
		padding: 32px 0 40px;
	}
	.logo {
		font-size: 17px;
		font-weight: 700;
		color: #1a1108;
		letter-spacing: -0.3px;
	}

	/* ── 전광판 ── */
	.ticker-wrap {
		overflow: hidden;
		background: linear-gradient(135deg, #1e2a1e, #111a11);
		border: 1px solid #2a3a2a;
		padding: 10px 0;
		box-shadow: inset 0 0 12px rgba(0,0,0,0.6);
		margin-bottom: 56px;
	}
	.ticker-track {
		display: flex;
		width: max-content;
		animation: ticker 28s linear infinite;
	}
	.ticker-track:hover { animation-play-state: paused; }
	.ticker-item {
		font-size: 0.72rem;
		font-weight: 500;
		letter-spacing: 0.1em;
		color: #90ee90;
		text-shadow: 0 0 8px rgba(144,238,144,0.55);
		white-space: nowrap;
		padding: 0 28px;
	}
	.ticker-sep {
		font-size: 0.72rem;
		color: #3a6a3a;
		white-space: nowrap;
		align-self: center;
	}
	.ticker-highlight .ticker-item {
		color: #b8f0b8;
		text-shadow: 0 0 10px rgba(144,238,144,0.8);
	}
	.ticker-warning .ticker-item {
		color: #ffb347;
		text-shadow: 0 0 10px rgba(255,140,0,0.8);
	}
	.ticker-warning .ticker-sep { color: #a0520a; }
	.ticker-static .ticker-track {
		animation: none;
		width: 100%;
		justify-content: center;
	}
	.ticker-static .ticker-item {
		animation: flash 1.2s ease-in-out infinite;
	}

	/* ── 메인 ── */
	.content {
		flex: 1;
		display: flex;
		flex-direction: column;
		gap: 48px;
	}

	/* ── 서비스 설명 ── */
	.intro {
		display: flex;
		flex-direction: column;
		gap: 16px;
	}
	.intro-title {
		font-size: 32px;
		font-weight: 700;
		line-height: 1.3;
		color: #1a1108;
		letter-spacing: -0.5px;
		margin: 0;
		word-break: keep-all;
	}
	.intro-desc {
		font-size: 16px;
		color: #313732;
		line-height: 1.8;
		margin: 0;
		word-break: keep-all;
	}

	/* ── 구독 폼 ── */
	.subscribe-form {
		display: flex;
		flex-direction: column;
		gap: 8px;
	}
	.input-row {
		display: flex;
		gap: 10px;
	}
	.email-input {
		flex: 1;
		padding: 12px 16px;
		font-size: 15px;
		border: 1px solid #e0e0e0;
		outline: none;
		border-radius: 8px;
		color: #1a1108;
		font-family: 'DM Sans', sans-serif;
		transition: border-color 0.15s;
	}
	.email-input:focus { border-color: #1a1108; }
	.email-input::placeholder { color: #aaa; }
	.subscribe-btn {
		padding: 12px 24px;
		background: #1a1108;
		color: #fff;
		font-size: 15px;
		font-weight: 600;
		border: none;
		border-radius: 8px;
		cursor: pointer;
		font-family: 'DM Sans', sans-serif;
		white-space: nowrap;
		transition: opacity 0.15s;
	}
	.subscribe-btn:hover:not(:disabled) { opacity: 0.8; }
	.subscribe-btn:disabled { opacity: 0.5; cursor: default; }

	.sent-msg {
		font-size: 14px;
		color: #313732;
		padding: 12px 0;
	}
	.error-msg {
		font-size: 13px;
		color: #c0392b;
	}

	/* ── 푸터 ── */
	.footer {
		padding: 40px 0 32px;
		font-size: 12px;
		color: #ccc;
		letter-spacing: 0.06em;
	}

	/* ── 해지 인라인 ── */
	.unsub-label {
		font-size: 12px;
		font-weight: 600;
		color: #889990;
		letter-spacing: 0.5px;
		text-transform: uppercase;
		margin: 0;
	}
	.cancel-link {
		font-size: 15px;
		color: #aaa;
		text-decoration: none;
		display: flex;
		align-items: center;
		transition: color 0.15s;
	}
	.cancel-link:hover { color: #333; }

	/* ── 애니메이션 ── */
	@keyframes ticker {
		from { transform: translateX(0); }
		to   { transform: translateX(-50%); }
	}
	@keyframes flash {
		0%, 100% { opacity: 1; text-shadow: 0 0 10px rgba(144,238,144,0.9); }
		50%       { opacity: 0.25; text-shadow: none; }
	}

	/* ── 모바일 ── */
	@media (max-width: 480px) {
		.input-row { flex-direction: column; }
		.subscribe-btn { width: 100%; }
	}
</style>
