<script>
	import { onMount, tick } from 'svelte';
	import { browser } from '$app/environment';

	let mapContainer;
	let map;
	let loading = true;
	let error = null;
	let mounted = false;

	const MAPBOX_TOKEN = import.meta.env.VITE_MAPBOX_ACCESS_TOKEN;

	// User data matching the image style - test
	const users = [
		{ name: 'City Trends', avatar: '👤', coords: [-73.9857, 40.7484], color: '#6366f1' },
		{ name: 'Alex Miller', avatar: '👨‍💼', coords: [-0.1276, 51.5074], color: '#8b5cf6' },
		{ name: 'Stanislav Kratochvil', avatar: '👨‍🦱', coords: [16.3738, 48.2082], color: '#06b6d4' },
		{ name: 'Jardins de Pruille', avatar: '🌱', coords: [2.3522, 48.8566], color: '#10b981' },
		{ name: 'Jardins de Pruille', avatar: '🌱', coords: [139.6917, 35.6895], color: '#f59e0b' },
		{ name: 'Maxime Goenvic', avatar: '👨‍💻', coords: [4.4699, 50.5039], color: '#ef4444' },
		{ name: 'Leon Arthur', avatar: '👨‍🎨', coords: [151.2093, -33.8688], color: '#8b5cf6' },
		{ name: 'Pedro de Almeida', avatar: '👨‍🏫', coords: [-46.6333, -23.5505], color: '#06b6d4' },
		{ name: 'Pedro de Almeida', avatar: '👨‍🏫', coords: [55.2708, 25.2048], color: '#10b981' }
	];

	onMount(async () => {
		if (!browser) return;

		try {
			if (!MAPBOX_TOKEN) {
				throw new Error(
					'Mapbox access token not found. Please add VITE_MAPBOX_ACCESS_TOKEN to your .env file.'
				);
			}

			await tick();

			if (!mapContainer) {
				throw new Error('Map container element is not available');
			}

			await loadMapboxFromCDN();

			const mapboxgl = window.mapboxgl;

			if (!mapboxgl || !mapboxgl.Map) {
				throw new Error('Mapbox GL JS failed to load properly');
			}

			mapboxgl.accessToken = MAPBOX_TOKEN;

			await new Promise((resolve) => setTimeout(resolve, 200));

			// Create map with light, clean style and ensure full viewport
			map = new mapboxgl.Map({
				container: mapContainer,
				style: 'mapbox://styles/mapbox/light-v11', // Clean light style
				projection: 'globe',
				zoom: 1.8,
				center: [20, 30],
				antialias: true,
				attributionControl: false, // Remove attribution for cleaner look
				logoPosition: 'bottom-right'
			});

			// Ensure map fills container completely
			map.getContainer().style.width = '100vw';
			map.getContainer().style.height = '100vh';

			// Force resize after map loads to ensure full viewport coverage
			map.on('load', () => {
				try {
					// Force map to resize to full viewport
					setTimeout(() => {
						map.resize();
						// Double-check container sizing
						const container = map.getContainer();
						container.style.position = 'fixed';
						container.style.top = '0';
						container.style.left = '0';
						container.style.width = '100vw';
						container.style.height = '100vh';
						container.style.zIndex = '1';
						map.resize();
					}, 100);

					// Add subtle atmosphere
					map.setFog({
						color: 'rgb(220, 231, 255)',
						'high-color': 'rgb(186, 210, 255)',
						'horizon-blend': 0.05,
						'space-color': 'rgb(240, 245, 255)',
						'star-intensity': 0.2
					});

					// Create user markers
					users.forEach((user, index) => {
						const el = document.createElement('div');
						el.className = 'user-marker';
						el.innerHTML = `
                <div class="marker-line"></div>
                <div class="marker-card" style="--user-color: ${user.color}">
                  <div class="marker-avatar">${user.avatar}</div>
                  <div class="marker-name">${user.name}</div>
                </div>
              `;

						new mapboxgl.Marker(el, { anchor: 'bottom' }).setLngLat(user.coords).addTo(map);
					});

					// Auto rotation
					let userInteracting = false;
					let spinEnabled = true;

					function spinGlobe() {
						if (spinEnabled && !userInteracting && map.getZoom() < 3) {
							const center = map.getCenter();
							center.lng -= 0.15;
							map.easeTo({
								center,
								duration: 1000,
								easing: (t) => t
							});
						}
					}

					map.on('mousedown', () => (userInteracting = true));
					map.on('mouseup', () => (userInteracting = false));
					map.on('dragend', () => (userInteracting = false));
					map.on('moveend', spinGlobe);

					spinGlobe();

					loading = false;
				} catch (setupError) {
					console.error('Error during map setup:', setupError);
					error = 'Error setting up globe features: ' + setupError.message;
					loading = false;
				}
			});

			map.on('error', (e) => {
				console.error('Map error:', e.error);
				error = `Map error: ${e.error?.message || 'Unknown error'}. Please check your Mapbox token.`;
				loading = false;
			});
		} catch (err) {
			console.error('Setup error:', err);
			error = err.message;
			loading = false;
		}

		mounted = true;
	});

	function loadMapboxFromCDN() {
		return new Promise((resolve, reject) => {
			if (window.mapboxgl) {
				resolve();
				return;
			}

			const cssLink = document.createElement('link');
			cssLink.rel = 'stylesheet';
			cssLink.href = 'https://api.mapbox.com/mapbox-gl-js/v2.15.0/mapbox-gl.css';
			document.head.appendChild(cssLink);

			const script = document.createElement('script');
			script.src = 'https://api.mapbox.com/mapbox-gl-js/v2.15.0/mapbox-gl.js';

			script.onload = () => {
				setTimeout(resolve, 100);
			};

			script.onerror = () => {
				reject(new Error('Failed to load Mapbox GL JS from CDN. Check your internet connection.'));
			};

			document.head.appendChild(script);
		});
	}
</script>

<div class="globe-wrapper">
	<!-- Map container always rendered -->
	<div
		bind:this={mapContainer}
		id="map"
		class="map-container"
		class:hidden={loading || error}
	></div>

	{#if loading}
		<div class="loading-overlay">
			<div class="loading-content">
				<div class="loading-spinner"></div>
				<h3>Loading Globe</h3>
				<p>Preparing your network visualization...</p>

				<div class="status-grid">
					<div class="status-item">
						<div class="status-dot {MAPBOX_TOKEN ? 'success' : 'error'}"></div>
						<span>Mapbox Token</span>
						<span class="status-text">{MAPBOX_TOKEN ? 'Found' : 'Missing'}</span>
					</div>

					<div class="status-item">
						<div class="status-dot {mounted ? 'success' : 'loading'}"></div>
						<span>Component</span>
						<span class="status-text">{mounted ? 'Ready' : 'Loading'}</span>
					</div>
				</div>
			</div>
		</div>
	{/if}

	{#if error}
		<div class="error-overlay">
			<div class="error-content">
				<div class="error-icon">⚠️</div>
				<h3>Unable to load globe</h3>
				<p class="error-message">{error}</p>

				<div class="setup-card">
					<h4>Quick Setup</h4>
					<div class="setup-steps">
						<div class="setup-step">
							<span class="step-number">1</span>
							<div>
								<strong>Create .env file</strong>
								<p>In your project root directory</p>
							</div>
						</div>

						<div class="setup-step">
							<span class="step-number">2</span>
							<div>
								<strong>Add your token</strong>
								<code>VITE_MAPBOX_ACCESS_TOKEN=pk.your_token_here</code>
							</div>
						</div>

						<div class="setup-step">
							<span class="step-number">3</span>
							<div>
								<strong>Get token from</strong>
								<a href="https://account.mapbox.com/access-tokens/" target="_blank"
									>Mapbox Dashboard</a
								>
							</div>
						</div>

						<div class="setup-step">
							<span class="step-number">4</span>
							<div>
								<strong>Restart server</strong>
								<code>npm run dev</code>
							</div>
						</div>
					</div>
				</div>
			</div>
		</div>
	{/if}

	{#if !loading && !error}
		<!-- Top header -->
		<div class="top-header">
			<div class="header-left">
				<div class="logo">
					<div class="logo-icon">🌐</div>
					<span class="logo-text">Network Globe</span>
				</div>
			</div>

			<div class="header-right">
				<div class="user-count">
					<span class="count-number">{users.length}</span>
					<span class="count-label">Active Users</span>
				</div>
			</div>
		</div>

		<!-- Bottom info -->
		<div class="bottom-info">
			<div class="info-card">
				<div class="info-header">
					<div class="live-indicator">
						<div class="live-dot"></div>
						<span>Live Network</span>
					</div>
				</div>
				<p class="info-description">
					Interactive user network visualization • Real-time global connections
				</p>
			</div>
		</div>
	{/if}
</div>

<style>
	/* Global reset to ensure full viewport */
	:global(html, body) {
		margin: 0 !important;
		padding: 0 !important;
		width: 100% !important;
		height: 100% !important;
		overflow: hidden !important;
	}

	:global(#svelte) {
		width: 100% !important;
		height: 100% !important;
		margin: 0 !important;
		padding: 0 !important;
	}

	/* Force absolute fullscreen positioning */
	.globe-wrapper {
		position: fixed !important;
		top: 0 !important;
		left: 0 !important;
		width: 100vw !important;
		height: 100vh !important;
		margin: 0 !important;
		padding: 0 !important;
		background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 50%, #cbd5e1 100%);
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
		overflow: hidden !important;
		z-index: 0;
	}

	.map-container {
		position: fixed !important;
		top: 0 !important;
		left: 0 !important;
		width: 100vw !important;
		height: 100vh !important;
		margin: 0 !important;
		padding: 0 !important;
		z-index: 1 !important;
	}

	.map-container.hidden {
		visibility: hidden !important;
		z-index: -1 !important;
	}

	/* Force Mapbox container to be fullscreen */
	:global(.mapboxgl-map) {
		position: fixed !important;
		top: 0 !important;
		left: 0 !important;
		width: 100vw !important;
		height: 100vh !important;
		margin: 0 !important;
		padding: 0 !important;
	}

	:global(.mapboxgl-canvas-container) {
		position: fixed !important;
		top: 0 !important;
		left: 0 !important;
		width: 100vw !important;
		height: 100vh !important;
	}

	:global(.mapboxgl-canvas) {
		position: fixed !important;
		top: 0 !important;
		left: 0 !important;
		width: 100vw !important;
		height: 100vh !important;
	}

	/* Loading overlay */
	.loading-overlay {
		position: absolute;
		inset: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		background: rgba(248, 250, 252, 0.95);
		backdrop-filter: blur(8px);
		z-index: 1000;
	}

	.loading-content {
		display: flex;
		flex-direction: column;
		align-items: center;
		text-align: center;
		gap: 1.5rem;
	}

	.loading-spinner {
		width: 2rem;
		height: 2rem;
		border: 2px solid #e2e8f0;
		border-top: 2px solid #3b82f6;
		border-radius: 50%;
		animation: spin 1s linear infinite;
	}

	.loading-content h3 {
		margin: 0;
		font-size: 1.25rem;
		font-weight: 600;
		color: #1e293b;
	}

	.loading-content p {
		margin: 0;
		font-size: 0.875rem;
		color: #64748b;
	}

	.status-grid {
		display: grid;
		gap: 0.75rem;
		min-width: 240px;
	}

	.status-item {
		display: flex;
		align-items: center;
		gap: 0.75rem;
		padding: 0.75rem;
		background: white;
		border: 1px solid #e2e8f0;
		border-radius: 0.5rem;
		font-size: 0.875rem;
	}

	.status-dot {
		width: 0.5rem;
		height: 0.5rem;
		border-radius: 50%;
	}

	.status-dot.success {
		background: #10b981;
	}
	.status-dot.error {
		background: #ef4444;
	}
	.status-dot.loading {
		background: #f59e0b;
	}

	.status-text {
		margin-left: auto;
		font-size: 0.75rem;
		color: #64748b;
	}

	/* Error overlay */
	.error-overlay {
		position: absolute;
		inset: 0;
		display: flex;
		align-items: center;
		justify-content: center;
		background: rgba(248, 250, 252, 0.95);
		backdrop-filter: blur(8px);
		z-index: 1000;
		padding: 2rem;
	}

	.error-content {
		max-width: 32rem;
		text-align: center;
	}

	.error-icon {
		font-size: 3rem;
		margin-bottom: 1rem;
	}

	.error-content h3 {
		margin: 0 0 0.5rem 0;
		font-size: 1.5rem;
		font-weight: 600;
		color: #dc2626;
	}

	.error-message {
		margin: 0 0 2rem 0;
		color: #64748b;
		line-height: 1.5;
	}

	.setup-card {
		background: white;
		border: 1px solid #e2e8f0;
		border-radius: 0.75rem;
		padding: 1.5rem;
		text-align: left;
	}

	.setup-card h4 {
		margin: 0 0 1rem 0;
		font-size: 1.125rem;
		font-weight: 600;
		color: #1e293b;
		text-align: center;
	}

	.setup-steps {
		display: flex;
		flex-direction: column;
		gap: 1rem;
	}

	.setup-step {
		display: flex;
		align-items: flex-start;
		gap: 0.75rem;
	}

	.step-number {
		width: 1.5rem;
		height: 1.5rem;
		background: #3b82f6;
		color: white;
		border-radius: 50%;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 0.75rem;
		font-weight: 600;
		flex-shrink: 0;
		margin-top: 0.125rem;
	}

	.setup-step strong {
		display: block;
		font-weight: 600;
		color: #1e293b;
		margin-bottom: 0.25rem;
	}

	.setup-step p {
		margin: 0;
		font-size: 0.875rem;
		color: #64748b;
	}

	.setup-step code {
		display: inline-block;
		background: #f1f5f9;
		border: 1px solid #e2e8f0;
		padding: 0.25rem 0.5rem;
		border-radius: 0.25rem;
		font-family: ui-monospace, monospace;
		font-size: 0.75rem;
		color: #3b82f6;
		margin-top: 0.25rem;
	}

	.setup-step a {
		color: #3b82f6;
		text-decoration: none;
	}

	.setup-step a:hover {
		text-decoration: underline;
	}

	/* Top header */
	.top-header {
		position: absolute;
		top: 0;
		left: 0;
		right: 0;
		display: flex;
		align-items: center;
		justify-content: space-between;
		padding: 1rem 1.5rem;
		background: rgba(255, 255, 255, 0.9);
		backdrop-filter: blur(8px);
		border-bottom: 1px solid rgba(226, 232, 240, 0.5);
		z-index: 100;
	}

	.header-left {
		display: flex;
		align-items: center;
	}

	.logo {
		display: flex;
		align-items: center;
		gap: 0.5rem;
	}

	.logo-icon {
		width: 2rem;
		height: 2rem;
		background: linear-gradient(135deg, #3b82f6, #1d4ed8);
		border-radius: 0.5rem;
		display: flex;
		align-items: center;
		justify-content: center;
		font-size: 1rem;
	}

	.logo-text {
		font-size: 1.125rem;
		font-weight: 600;
		color: #1e293b;
	}

	.header-right {
		display: flex;
		align-items: center;
		gap: 1rem;
	}

	.user-count {
		display: flex;
		flex-direction: column;
		align-items: flex-end;
	}

	.count-number {
		font-size: 1.5rem;
		font-weight: 700;
		color: #3b82f6;
		line-height: 1;
	}

	.count-label {
		font-size: 0.75rem;
		color: #64748b;
		text-transform: uppercase;
		letter-spacing: 0.05em;
	}

	/* Bottom info */
	.bottom-info {
		position: absolute;
		bottom: 1.5rem;
		left: 1.5rem;
		z-index: 100;
	}

	.info-card {
		background: rgba(255, 255, 255, 0.95);
		backdrop-filter: blur(8px);
		border: 1px solid rgba(226, 232, 240, 0.5);
		border-radius: 0.75rem;
		padding: 1rem;
		min-width: 280px;
	}

	.info-header {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		margin-bottom: 0.5rem;
	}

	.live-indicator {
		display: flex;
		align-items: center;
		gap: 0.5rem;
		font-size: 0.875rem;
		font-weight: 600;
		color: #1e293b;
	}

	.live-dot {
		width: 0.5rem;
		height: 0.5rem;
		background: #10b981;
		border-radius: 50%;
		animation: pulse 2s infinite;
	}

	.info-description {
		margin: 0;
		font-size: 0.875rem;
		color: #64748b;
		line-height: 1.4;
	}

	/* User markers */
	:global(.user-marker) {
		position: relative;
		cursor: pointer;
	}

	:global(.marker-line) {
		position: absolute;
		bottom: 0;
		left: 50%;
		width: 1px;
		height: 2rem;
		background: linear-gradient(to bottom, transparent, rgba(59, 130, 246, 0.3));
		transform: translateX(-50%);
	}

	:global(.marker-card) {
		position: absolute;
		bottom: 2rem;
		left: 50%;
		transform: translateX(-50%);
		background: white;
		border: 1px solid #e2e8f0;
		border-radius: 0.5rem;
		padding: 0.5rem 0.75rem;
		min-width: 120px;
		text-align: center;
		box-shadow:
			0 4px 6px -1px rgba(0, 0, 0, 0.1),
			0 2px 4px -1px rgba(0, 0, 0, 0.06);
		transition: all 0.2s ease;
		border-left: 3px solid var(--user-color);
	}

	:global(.marker-card:hover) {
		transform: translateX(-50%) translateY(-4px);
		box-shadow:
			0 10px 15px -3px rgba(0, 0, 0, 0.1),
			0 4px 6px -2px rgba(0, 0, 0, 0.05);
	}

	:global(.marker-avatar) {
		font-size: 1.25rem;
		margin-bottom: 0.25rem;
	}

	:global(.marker-name) {
		font-size: 0.75rem;
		font-weight: 600;
		color: #1e293b;
		line-height: 1.2;
	}

	/* Animations */
	@keyframes spin {
		from {
			transform: rotate(0deg);
		}
		to {
			transform: rotate(360deg);
		}
	}

	@keyframes pulse {
		0%,
		100% {
			opacity: 1;
		}
		50% {
			opacity: 0.5;
		}
	}

	/* Responsive */
	@media (max-width: 768px) {
		.top-header {
			padding: 0.75rem 1rem;
		}

		.bottom-info {
			bottom: 1rem;
			left: 1rem;
			right: 1rem;
		}

		.info-card {
			min-width: auto;
		}

		.logo-text {
			display: none;
		}

		.user-count {
			align-items: center;
		}
	}
</style>
