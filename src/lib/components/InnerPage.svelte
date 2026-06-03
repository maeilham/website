<script lang="ts">
	let { children } = $props();
</script>

<div class="shell">
	<header class="header">
		<a href="/" class="logo">
			<span class="logo-icon">✉</span>
			<span class="logo-text">매일함</span>
		</a>
	</header>

	<main class="content">
		{@render children()}
	</main>

	<footer class="footer">
		매일함으로, 매일 함.
	</footer>
</div>

<style>
	.shell {
		min-height: 100vh;
		display: flex;
		flex-direction: column;
		background: #fff;
		color: #111;
		font-family: 'DM Sans', sans-serif;
	}

	.header {
		padding: 20px 32px;
		border-bottom: 1px solid #f0f0f0;
	}

	.logo {
		display: inline-flex;
		align-items: center;
		gap: 8px;
		text-decoration: none;
	}

	.logo-icon {
		font-size: 0.9rem;
		color: #999;
	}

	.logo-text {
		font-size: 0.95rem;
		font-weight: 500;
		color: #111;
		letter-spacing: -0.01em;
		font-family: 'DM Sans', sans-serif;
	}

	.content {
		flex: 1;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 64px 24px;
	}

	.footer {
		padding: 20px 32px;
		font-size: 0.7rem;
		color: #ccc;
		letter-spacing: 0.06em;
	}
</style>
