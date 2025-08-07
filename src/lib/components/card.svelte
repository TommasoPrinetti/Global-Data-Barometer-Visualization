<script lang="ts">
	import gdBarometerLogo from '$lib/assets/globaldatabarometer.png';
	import QuestionMark from '$lib/components/question_mark.svelte';
	import BarChart from '$lib/components/BarChart.svelte';
	import {
		isChartMode,
		isWorstMode,
		isCurrentCountry,
		globalOverview,
		threePillars,
		evaluationClusters
	} from '$lib/stores.js';
	import { fade, scale } from 'svelte/transition';
	import html2canvas from 'html2canvas';
	import { onMount } from 'svelte';
	import { interactivity } from '@threlte/extras';
	import { writable } from 'svelte/store';
	export let countryData: any;
	export let isSmall = false;
	export let index: number;
	export let countryImages: any;

	let showSharer = writable(false);
	let copierAlert: HTMLElement;
	let cardContainer: HTMLElement | null;
	let terrainBox: HTMLElement | null;
	let terrainBoxWidth: number = 0;
	let terrainBoxHeight: number = 0;

	function getCountryImageKey(countryName: string): string {
		const countryMapping: { [key: string]: string } = {
			'Burkina Faso': 'burkinaFaso',
			'Costa Rica': 'costaRica',
			Cote_D_Ivoire: 'coteDIvoire',
			'Dominican Republic': 'dominicanRepublic',
			'El Salvador': 'elSalvador',
			'Sierra Leone': 'sierraLeone',
			'South Africa': 'southAfrica',
			'Trinidad and Tobago': 'trinidadAndTobago'
		};

		return countryMapping[countryName] || countryName.toLowerCase().replace(/\s+/g, '');
	}

	async function exportAsJPG() {
		if (!cardContainer) return;

		const questionMarks = cardContainer.querySelectorAll('.question_mark_svg');
		const downloadButton = cardContainer.querySelector('.download_button');

		const prevDisplay: string[] = [];

		questionMarks.forEach((el, i) => {
			prevDisplay[i] = (el as HTMLElement).style.display;
			(el as HTMLElement).style.display = 'none';
		});

		if (downloadButton) {
			prevDisplay[0] = (downloadButton as HTMLElement).style.display;
			(downloadButton as HTMLElement).style.display = 'none';
		}

		try {
			const cardCanvas = await html2canvas(cardContainer, {
				backgroundColor: 'transparent',
				scale: 2,
				useCORS: true,
				allowTaint: true,
				imageTimeout: 15000
			});

			const finalCanvas = document.createElement('canvas');
			const ctx = finalCanvas.getContext('2d');

			finalCanvas.width = 2600;
			finalCanvas.height = 1600;

			if (ctx) {
				ctx.fillStyle = '#ffffff';
				ctx.fillRect(0, 0, finalCanvas.width, finalCanvas.height);

				const cardWidth = cardCanvas.width;
				const cardHeight = cardCanvas.height;
				const bottomPadding = 100;
				const x = (finalCanvas.width - cardWidth) / 2;
				const y = (finalCanvas.height - cardHeight - bottomPadding) / 2;

				ctx.drawImage(cardCanvas, x, y);

				const logoImg = new Image();
				logoImg.crossOrigin = 'anonymous';
				logoImg.onload = () => {
					const logoWidth = 400;
					const logoHeight = (logoWidth * logoImg.height) / logoImg.width;
					const logoX = (finalCanvas.width - logoWidth) / 2;
					const logoY = finalCanvas.height - bottomPadding - 30;

					ctx.drawImage(logoImg, logoX, logoY, logoWidth, logoHeight);

					const url = finalCanvas.toDataURL('image/jpeg', 0.9);
					const link = document.createElement('a');
					link.href = url;
					link.download = `${countryData.CountryName}_card.jpg`;
					document.body.appendChild(link);
					link.click();
					document.body.removeChild(link);
				};
				logoImg.src = gdBarometerLogo;
			}
		} catch (error) {
			console.error('Export error:', error);
		} finally {
			questionMarks.forEach((el, i) => {
				(el as HTMLElement).style.display = prevDisplay[i] || '';
			});

			if (downloadButton) {
				(downloadButton as HTMLElement).style.display = prevDisplay[0] || '';
			}
		}
	}

	async function shareOnTwitter() {
		await exportAsJPG();
		const header = document.querySelector('.sharer_header h2');
		if (header) {
			header.textContent = 'Image downloaded successfully!';
		}
		setTimeout(() => {
			const text = encodeURIComponent(
				`Check out how ${countryData.CountryName} performed in 2024 Global Data Barometer results!`
			);
			const url = encodeURIComponent(window.location.href);
			window.open(`https://twitter.com/intent/tweet?text=${text}&url=${url}`);
		}, 1500);
	}

	async function shareOnReddit() {
		await exportAsJPG();
		const header = document.querySelector('.sharer_header h2');
		if (header) {
			header.textContent = 'Image downloaded successfully!';
		}
		setTimeout(() => {
			const title = encodeURIComponent(`${countryData.CountryName} Data Barometer`);
			const url = encodeURIComponent(window.location.href);
			window.open(`https://reddit.com/submit?url=${url}&title=${title}`);
		}, 1500);
	}

	async function shareOnBluesky() {
		await exportAsJPG();
		const header = document.querySelector('.sharer_header h2');
		if (header) {
			header.textContent = 'Image downloaded successfully!';
		}
		setTimeout(() => {
			const text = encodeURIComponent(
				`Check out ${countryData.CountryName}'s data barometer results! ${window.location.href}`
			);
			window.open(`https://bsky.app/intent/compose?text=${text}`);
		}, 1500);
	}

	async function downloadImage() {
		exportAsJPG();
	}

	function toggleCard(el: HTMLElement) {
		if (el.classList.contains('hidden')) {
			el.classList.remove('hidden');
		} else {
			el.style.transition = 'transform 1.65s cubic-bezier(1,0,.63,1.01)';
			el.classList.add('hidden');
			setTimeout(() => {
				isCurrentCountry.set('');
			}, 1450);
		}
	}

	onMount(() => {
		if (isSmall) {
			interactivity();
		}

		if (terrainBox) {
			const rect = terrainBox.getBoundingClientRect();
			terrainBoxWidth = rect.width;
			terrainBoxHeight = rect.height;
		}
	});
</script>

{#if $isCurrentCountry === countryData.CountryName && !isSmall && countryImages}
	<script>
		setTimeout(() => {
			if (document.querySelector(`.card_container.hidden`)) {
				document.querySelector(`.card_container.hidden`).classList.remove('hidden');
			} else {
				document.querySelector(`.card_container`).classList.add('hidden');
			}
		}, 600);
	</script>
	<div
		class="card_container hidden"
		aria-label="Country data card"
		id={`card_${countryData.CountryName}`}
		bind:this={cardContainer}
		out:scale={{ duration: 1000, delay: 1000, easing: (t) => t * t }}
	>
		{#if !$isChartMode}
			<div class="card_columns" id="terrain_column" bind:this={terrainBox}>
				{#if countryImages[getCountryImageKey(countryData.CountryName)]}
					<img
						in:fade={{ duration: 1000, delay: 1000, easing: (t) => t * t }}
						src={countryImages[getCountryImageKey(countryData.CountryName)]}
						alt={countryData.CountryName}
						class="terrain_image"
					/>
				{/if}
			</div>

			<div class="card_columns" id="textual_data">
				<div>
					<div class="data_header">
						<div>
							<h2>{countryData.CountryName}</h2>
							<p>{countryData.Flag}</p>
						</div>
						<button
							class="download_button"
							aria-label="Download country data"
							onclick={() => showSharer.set(true)}
						>
							<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960"
								><path
									d="M242.87-30.43q-37.78 0-64.39-26.61t-26.61-64.39v-423.74q0-37.79 26.61-64.4 26.61-26.6 64.39-26.6H354.5v91H242.87v423.74h474.26v-423.74H605.5v-91h111.63q37.78 0 64.39 26.6 26.61 26.61 26.61 64.4v423.74q0 37.78-26.61 64.39t-64.39 26.61H242.87Zm191.63-281.2v-444.13l-64 64-63.65-64.41L480-929.57l173.15 173.4-63.65 64.41-64-64v444.13h-91Z"
								/></svg
							>
						</button>
					</div>
					<div class="data_grid">
						<div>
							<p class="data_title">Global overview</p>
							<div class="data_triple_grid">
								{#each globalOverview as item}
									<div class="number_container">
										<h3>{item.displayName}<QuestionMark type={item.displayName} /></h3>
										<p>{countryData[item.dataKey]}<span>/{item.maxValue}</span></p>
									</div>
								{/each}
							</div>
						</div>
						<div>
							<p class="data_title">Three main pillars</p>
							<div class="data_triple_grid">
								{#each threePillars as pillar}
									<div class="number_container">
										<h3>{pillar.displayName}<QuestionMark type={pillar.displayName} /></h3>
										<p>{countryData[pillar.dataKey]}<span>/{pillar.maxValue}</span></p>
									</div>
								{/each}
							</div>
						</div>
						<div>
							<p class="data_title">Eight evaluation clusters</p>
							<div class="data_triple_grid">
								{#each evaluationClusters as cluster}
									<div class="number_container">
										<h3>{cluster.displayName}<QuestionMark type={cluster.displayName} /></h3>
										<p>{countryData[cluster.dataKey]}<span>/{cluster.maxValue}</span></p>
									</div>
								{/each}
							</div>
						</div>
					</div>
				</div>
				<img src={gdBarometerLogo} alt="GD Barometer logo" class="gd_barometer_logo" />
			</div>
		{:else if $isChartMode}
			<div class="chart_grandpa">
				<div class="data_header">
					<div>
						<h2>{countryData.CountryName}</h2>
					</div>
					<p style="align-self: center; font-size: 52px;">{countryData.Flag}</p>
					<button
						class="download_button"
						aria-label="Download country data"
						onclick={() => showSharer.set(true)}
					>
						<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960"
							><path
								d="M242.87-30.43q-37.78 0-64.39-26.61t-26.61-64.39v-423.74q0-37.79 26.61-64.4 26.61-26.6 64.39-26.6H354.5v91H242.87v423.74h474.26v-423.74H605.5v-91h111.63q37.78 0 64.39 26.6 26.61 26.61 26.61 64.4v423.74q0 37.78-26.61 64.39t-64.39 26.61H242.87Zm191.63-281.2v-444.13l-64 64-63.65-64.41L480-929.57l173.15 173.4-63.65 64.41-64-64v444.13h-91Z"
							/></svg
						>
					</button>
				</div>
				<div class="chart_header">
					<div class="number_container">
						<h3>Population<QuestionMark type="Population" /></h3>
						<p>{countryData.Population}M people</p>
					</div>
					<div class="number_container">
						<h3>Gov. form<QuestionMark type="Gov. form" /></h3>
						<p style="white-space: nowrap;">{countryData.GovForm}</p>
					</div>
					<div class="number_container">
						<h3>GDP<QuestionMark type="GDP" /></h3>
						<p>${countryData.Gdp} Million</p>
					</div>
					{#each globalOverview as item}
						<div class="number_container">
							<h3>{item.displayName}<QuestionMark type={item.displayName} /></h3>
							<p>{countryData[item.dataKey]}<span>/{item.maxValue}</span></p>
						</div>
					{/each}
				</div>
				<div class="double_column_chart">
					<div class="single_column_chart">
						<div class="column_header">
							<h2>Three main pillars <QuestionMark type="pillars" /></h2>
							<button class="worst_best_button" onclick={() => isWorstMode.set(!$isWorstMode)}>
								{#if $isWorstMode}
									<p>w/ best country</p>
								{:else}
									<p>w/ worst country</p>
								{/if}
							</button>
						</div>
						<div class="sx_column">
							{#each threePillars as pillar}
								<div class="vertical_chart_container">
									<BarChart
										length={countryData[pillar.dataKey]}
										maxValue={$isWorstMode ? pillar.worstValue : pillar.maxValue}
										flag={countryData.Flag}
										isVertical={true}
									/>
									<div class="number_container">
										<h3>{pillar.displayName}<QuestionMark type={pillar.displayName} /></h3>
										<p>{countryData[pillar.dataKey]}<span>/{pillar.maxValue}</span></p>
									</div>
								</div>
							{/each}
						</div>
					</div>
					<div class="single_column_chart">
						<div class="column_header">
							<h2>Eight evaluation clusters<QuestionMark type="clusters" /></h2>
							<button class="worst_best_button" onclick={() => isWorstMode.set(!$isWorstMode)}>
								{#if $isWorstMode}
									<p>w/ best country</p>
								{:else}
									<p>w/ worst country</p>
								{/if}
							</button>
						</div>
						<div class="dx_column">
							{#each evaluationClusters as cluster}
								<div class="single_chart_container">
									<div class="number_container">
										<h3>{cluster.displayName}<QuestionMark type={cluster.displayName} /></h3>
										<p>{countryData[cluster.dataKey]}<span>/{cluster.maxValue}</span></p>
									</div>
									<BarChart
										length={countryData[cluster.dataKey]}
										maxValue={$isWorstMode ? cluster.worstValue : cluster.maxValue}
										flag={countryData.Flag}
										isVertical={false}
									/>
								</div>
							{/each}
						</div>
					</div>
				</div>
			</div>
		{/if}

		<button
			class="side_button"
			aria-label="Close"
			onclick={() => cardContainer && toggleCard(cardContainer)}
		>
			<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960"
				><path
					d="M256-192.35 192.35-256l224-224-224-224L256-767.65l224 224 224-224L767.65-704l-224 224 224 224L704-192.35l-224-224-224 224Z"
				/></svg
			>
		</button>

		<button
			class="side_button"
			aria-label="Swap"
			onclick={() => {
				isChartMode.set(!$isChartMode);
			}}
		>
			<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960"
				><path
					d="M481.52-154.26q-135.43 0-231.23-94.2-95.79-94.19-95.79-228.91v-7.24L94.09-424.2 35.7-482.83 200-647.37l164.3 164.54-58.39 58.63-60.41-60.41v6.52q0 96.89 68.83 164.86 68.82 67.97 167.19 67.97 26.72 0 52.32-6.36 25.59-6.36 50.07-19.08l67.42 67.18q-39.68 24.39-82.31 36.82-42.63 12.44-87.5 12.44Zm278.72-158.13L595.7-476.7l58.63-58.63 60.41 60.42v-6.52q0-96.9-68.95-164.86-68.94-67.97-167.31-67.97-26.72 0-52.32 6.36-25.59 6.36-50.07 19.07l-67.18-67.41q39.68-24.39 82.19-36.71 42.51-12.31 87.38-12.31 135.43 0 231.35 94.19 95.91 94.2 95.91 228.92v7.24l60.17-60.42 58.63 58.63-164.3 164.31Z"
				/></svg
			>
		</button>
	</div>
{:else if isSmall && countryImages}
	<div class="small_card_sphere" style="transition-delay: {index * 0.1}s;"></div>
	<div
		class="small_card_container"
		style="transition-delay: {index * 0.1 + 1.5}s;"
		role="button"
		tabindex="0"
		onpointerenter={(e) => {
			const element = (e.target as HTMLElement).closest('.small_card_container') as HTMLElement;
			const siblings = Array.from(document.querySelectorAll('.small_card_container')).filter(
				(child) => child !== element
			);
			const yellowSpheres = document.querySelectorAll(
				'.small_card_sphere'
			) as NodeListOf<HTMLElement>;
			element.style.transform = 'scale(15)';
			element.style.zIndex = '1000';
			element.style.opacity = '1';
			element.style.transition = 'transform 0.65s ease-in-out';
			element.style.transitionDelay = '0.1s';
			siblings.forEach((sibling) => {
				(sibling as HTMLElement).style.transform = 'scale(0)';
				(sibling as HTMLElement).style.pointerEvents = 'none';
				(sibling as HTMLElement).style.transition = 'transform 0.65s ease-in-out';
			});
			yellowSpheres.forEach((sphere) => {
				sphere.style.transform = 'scale(0)';
				sphere.style.transition = 'transform 0.65s ease-in-out';
			});
		}}
		onpointerleave={(e) => {
			const element = (e.target as HTMLElement).closest('.small_card_container') as HTMLElement;
			const siblings = Array.from(document.querySelectorAll('.small_card_container')).filter(
				(child) => child !== element
			);
			const yellowSpheres = document.querySelectorAll(
				'.small_card_sphere'
			) as NodeListOf<HTMLElement>;
			element.style.transform = 'scale(5)';
			element.style.zIndex = '1';
			element.style.transition = 'transform 0.65s ease-in-out';
			element.style.transitionDelay = '0.1s';
			siblings.forEach((sibling) => {
				(sibling as HTMLElement).style.transform = 'scale(7)';
				(sibling as HTMLElement).style.pointerEvents = 'all';
				(sibling as HTMLElement).style.transition = 'transform 0.65s ease-in-out';
			});
			yellowSpheres.forEach((sphere) => {
				sphere.style.transform = 'scale(7)';
				sphere.style.transition = 'transform 0.65s ease-in-out';
			});
		}}
	>
		<div class="small_map">
			{#if countryImages[getCountryImageKey(countryData.CountryName)]}
				<img
					src={countryImages[getCountryImageKey(countryData.CountryName)]}
					alt="Map"
					loading="lazy"
				/>
			{/if}
		</div>
		<div class="small_info_container">
			<div class="small_header">
				<h2>{countryData.CountryName}</h2>
				<p>{countryData.Flag}</p>
			</div>
			<div class="small_counters">
				<p><b>Average:</b> {countryData.Overall}</p>
				<p><b>Availability:</b> {countryData.Availability}</p>
				<p><b>Capability:</b> {countryData.Capability}</p>
			</div>
		</div>
	</div>
{/if}

{#if $showSharer}
	<div
		class="sharer_global_container"
		in:scale={{ duration: 400, delay: 200, start: 0, opacity: 0 }}
		out:scale={{ duration: 400, delay: 200, start: 0, opacity: 0 }}
		aria-modal="true"
		role="dialog"
		tabindex="0"
	>
		<div class="sharer_header">
			<h2>Share this card on:</h2>
			<button onclick={() => showSharer.set(false)} id="closer" aria-label="Close share dialog">
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960"
					><path
						d="M256-227.69 227.69-256l224-224-224-224L256-732.31l224 224 224-224L732.31-704l-224 224 224 224L704-227.69l-224-224-224 224Z"
					/></svg
				>
			</button>
		</div>

		<div class="sharer_logo_container">
			<button onclick={shareOnTwitter} id="twitter_sharer" aria-label="Share on Twitter">
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 280 280">
					<path
						d="M166.665 118.575L270.919 0H246.223L155.661 102.936L83.384 0H0L109.321 155.672L0 280H24.696L120.269 171.273L196.616 280H280M33.6093 18.2221H71.5493L246.204 262.673H208.255"
					/>
				</svg>
			</button>
			<button id="reddit_sharer" onclick={shareOnReddit} aria-label="Share on Reddit">
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 280 280">
					<path
						d="M140.077 186.577C151.036 186.564 161.989 187.102 172.895 188.188C173.303 188.228 173.696 188.363 174.043 188.582C174.39 188.801 174.68 189.099 174.892 189.45C175.104 189.801 175.23 190.197 175.263 190.605C175.295 191.014 175.231 191.424 175.077 191.804C172.193 198.709 167.332 204.607 161.104 208.756C154.876 212.904 147.56 215.119 140.077 215.119C132.593 215.119 125.277 212.905 119.049 208.756C112.821 204.607 107.96 198.709 105.077 191.804C104.92 191.424 104.854 191.013 104.885 190.604C104.916 190.194 105.042 189.797 105.254 189.446C105.467 189.094 105.758 188.796 106.106 188.578C106.454 188.359 106.85 188.226 107.259 188.188C117.479 187.15 128.538 186.577 140.077 186.577Z"
					/>
					<path
						d="M96.8749 138.452C106.045 138.452 113.034 148.066 112.485 159.931V159.942C111.937 171.795 104.073 176.1 94.9034 176.101C85.7217 176.101 78.7445 170.034 79.2928 158.169C79.8414 146.304 87.7166 138.452 96.8749 138.452Z"
					/>
					<path
						d="M183.278 138.452C192.436 138.453 200.312 146.304 200.86 158.169C201.409 170.034 194.419 176.101 185.261 176.101C176.091 176.1 168.229 171.795 167.68 159.931C167.132 148.078 174.12 138.452 183.278 138.452Z"
					/>
					<path
						fill-rule="evenodd"
						clip-rule="evenodd"
						d="M140 0C217.327 0 280 62.6733 280 140C280 217.327 217.327 280 140 280H20.2769C18.6149 280.002 16.9892 279.51 15.6065 278.588C14.2243 277.666 13.1468 276.354 12.5099 274.82C11.8729 273.285 11.7051 271.595 12.0286 269.965C12.3521 268.335 13.1521 266.838 14.3269 265.662L41.0089 238.991C27.9899 226.005 17.6647 210.575 10.6272 193.587C3.58988 176.599 -0.0219603 158.388 0.000100451 140C0.000100451 62.6734 62.6735 0.000105513 140 0ZM198.609 42.1066C194.459 40.7071 189.998 40.5127 185.742 41.5447C181.487 42.5768 177.611 44.7932 174.564 47.9377C171.517 51.0824 169.423 55.0268 168.525 59.3127C159.241 60.275 150.644 64.6492 144.4 71.5873C138.156 78.5253 134.709 87.534 134.727 96.868C114.229 97.7313 95.4919 103.565 80.6286 112.793C76.9907 109.995 72.804 107.994 68.3417 106.921C63.8793 105.848 59.2411 105.726 54.7286 106.564C50.2161 107.402 45.93 109.181 42.1505 111.785C38.3713 114.389 35.1826 117.759 32.7921 121.677C30.4015 125.595 28.8631 129.972 28.2763 134.524C27.6894 139.076 28.0677 143.702 29.3864 148.098C30.7052 152.494 32.9344 156.563 35.9298 160.04C38.9252 163.517 42.6193 166.325 46.7716 168.28C47.8568 206.232 89.2155 236.752 140.082 236.752C190.984 236.752 232.365 206.185 233.391 168.198C237.522 166.225 241.194 163.406 244.166 159.924C247.138 156.442 249.346 152.374 250.647 147.984C251.947 143.595 252.309 138.98 251.713 134.441C251.116 129.902 249.573 125.539 247.183 121.635C244.793 117.73 241.61 114.37 237.839 111.774C234.068 109.178 229.793 107.403 225.292 106.564C220.791 105.725 216.165 105.842 211.712 106.904C207.259 107.967 203.078 109.954 199.441 112.736C184.438 103.449 165.491 97.6038 144.772 96.8338V96.7518C144.772 82.7984 155.19 71.2245 168.654 69.4162C169.859 74.4904 172.741 79.0102 176.833 82.2432C180.926 85.4761 185.99 87.2341 191.205 87.232C195.584 87.2317 199.873 85.9897 203.576 83.6514C207.278 81.313 210.243 77.9736 212.125 74.0195C214.007 70.0657 214.729 65.6594 214.21 61.3115C213.69 56.9634 211.948 52.8514 209.187 49.4525C206.425 46.0537 202.758 43.5062 198.609 42.1066Z"
					/>
				</svg>
			</button>
			<button id="download_sharer" onclick={downloadImage} aria-label="Download image">
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960"
					><path
						d="m481.07-394.98-57.59 57.59 60.3 60.3 161-161-161-161-60.3 60.31 57.59 57.58H317.13v86.22h163.94Zm-318.2 243.11q-37.78 0-64.39-26.61t-26.61-64.39v-474.26q0-37.78 26.61-64.39t64.39-26.61h233.54L480-724.54h317.13q37.78 0 64.39 26.61 26.61 26.6 26.61 64.39v390.67q0 37.78-26.61 64.39t-64.39 26.61H162.87Zm0-91h634.26v-390.67H442.46l-83.59-83.59h-196v474.26Zm0 0v-474.26 474.26Z"
					/></svg
				>
			</button>
			<button id="bluesky_sharer" onclick={shareOnBluesky} aria-label="Share on Bluesky">
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 318 280">
					<path
						d="M68.9307 18.8474C105.389 46.2185 144.603 101.716 159.001 131.499C173.4 101.718 212.612 46.218 249.071 18.8474C275.377 -0.902434 318 -16.1838 318 32.4422C318 42.1535 312.432 114.022 309.167 125.69C297.816 166.254 256.454 176.6 219.661 170.338C283.974 181.284 300.334 217.541 265.002 253.799C197.899 322.658 168.555 236.522 161.033 214.45C159.654 210.404 159.009 208.511 159 210.121C158.99 208.511 158.346 210.404 156.967 214.45C149.448 236.522 120.105 322.66 52.9982 253.799C17.6653 217.541 34.0253 181.282 98.3387 170.338C61.5452 176.6 20.1825 166.254 8.83322 125.69C5.5676 114.021 0 42.1524 0 32.4422C0 -16.1838 42.6238 -0.902434 68.9288 18.8474H68.9307Z"
					/>
				</svg>
			</button>
		</div>
		<div class="sharer_link_container">
			<div class="sharer_link_text">
				<p class="p3">
					{window.location.href}
				</p>
			</div>
			<button
				class="sharer_copier"
				onclick={() => {
					navigator.clipboard.writeText(window.location.href);
					copierAlert.classList.add('show');
					setTimeout(() => {
						copierAlert.classList.remove('show');
					}, 1000);
				}}
				aria-label="Copy link to clipboard"
			>
				<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 -960 960 960"
					><path
						d="M362.31-260Q332-260 311-281q-21-21-21-51.31v-455.38Q290-818 311-839q21-21 51.31-21h335.38Q728-860 749-839q21 21 21 51.31v455.38Q770-302 749-281q-21 21-51.31 21H362.31Zm0-60h335.38q4.62 0 8.46-3.85 3.85-3.84 3.85-8.46v-455.38q0-4.62-3.85-8.46-3.84-3.85-8.46-3.85H362.31q-4.62 0-8.46 3.85-3.85 3.84-3.85 8.46v455.38q0 4.62 3.85 8.46 3.84 3.85 8.46 3.85Zm-140 200Q192-120 171-141q-21-21-21-51.31v-515.38h60v515.38q0 4.62 3.85 8.46 3.84 3.85 8.46 3.85h395.38v60H222.31ZM350-320v-480 480Z"
					/></svg
				>
				<div class="copier_alert" bind:this={copierAlert}>
					<p class="p3">Link copied!</p>
				</div>
			</button>
		</div>
	</div>
	<div
		class="sharer_background"
		in:fade={{ duration: 600 }}
		out:fade={{ duration: 600 }}
		onclick={() => showSharer.set(false)}
		onkeydown={(e) => e.key === 'Enter' && showSharer.set(false)}
		role="button"
		tabindex="0"
	></div>
{/if}

<style>
	* {
		font-family: 'Rethink Sans', sans-serif;
		box-sizing: border-box;
		margin: 0;
		padding: 0;
		letter-spacing: -4%;
	}

	b {
		font-weight: 600;
	}

	.side_button {
		position: absolute;
		fill: white;
		width: 45px;
		height: 45px;
		border-radius: 100%;
		background-color: black;
		padding: 10px;
		right: -5%;
		top: 0;
		cursor: pointer;
		transition: transform 0.3s ease-in-out;
	}

	.side_button:hover {
		background-color: rgba(23, 23, 23, 0.5);
		transition: transform 0.1s ease-in-out;
	}

	.side_button:active {
		transform: scale(0.95);
		transition: transform 0.1s ease-in-out;
	}

	:global(.terrain_image) {
		max-width: 100%;
		max-height: 100%;
		width: auto;
		height: auto;
		object-fit: contain;
		object-position: center;
	}

	:global(.terrain_image picture) {
		width: 100%;
		height: 100%;
	}

	:global(.terrain_image > picture > img) {
		width: 100%;
		height: 100%;
		object-fit: contain;
		max-width: 100%;
		max-height: 100%;
	}

	.side_button > svg {
		width: 100%;
		height: 100%;
	}

	.side_button:nth-of-type(2) {
		transform: translateY(110%) scale(1);
		background-color: rgb(255, 239, 65);
		fill: black;
		border: 1px solid black;
		transition: transform 0.8s ease-in-out filter 0.3s ease-in-out background-color 0.3s ease-in-out;
	}

	.side_button:nth-of-type(2):hover {
		background-color: rgb(192, 180, 48);
		transform: translateY(110%) scale(1) rotate(180deg);
		transition: transform 0.8s ease-in-out filter 0.3s ease-in-out background-color 0.3s ease-in-out;
	}

	.side_button:nth-of-type(2):active {
		transform: translateY(110%) scale(0.95);
		transition: transform 2s ease-in-out filter 0.1s ease-in-out;
	}

	.download_button {
		fill: rgb(2, 2, 2);
		width: 30px;
		height: 30px;
		transition: transform 0.3s ease-in-out;
	}

	.gd_barometer_logo {
		width: 150px;
		align-self: flex-end;
	}

	.chart_grandpa > .gd_barometer_logo {
		position: absolute;
		bottom: 3%;
		right: 2%;
		display: block;
		width: 120px;
		border-radius: 20px;
	}

	.download_button:active {
		fill: #595959;
		transform: scale(0.95);
		transition: transform 0.1s ease-in-out;
	}

	.card_container {
		position: absolute;
		bottom: 50%;
		left: 50%;
		transform: translate(-50%, 55%) scale(0.8);
		background-color: #dddddd;
		border: 1px solid #000000;
		display: flex;
		flex-direction: row;

		width: 1300px;
		min-width: 920px;
		max-height: 1000px;

		height: 90%;

		justify-content: space-between;
		border-radius: 30px;
		padding: 20px;
		column-gap: 20px;
		z-index: 10;
		color: black;

		transition:
			transform 1s cubic-bezier(0.165, 0.84, 0.44, 1),
			opacity 0.4s cubic-bezier(0.165, 0.84, 0.44, 1);

		transition-delay: 0.6s;
		will-change: transform, opacity;
	}

	@media (min-width: 1600px) {
		.card_container {
			width: 1600px;
		}

		#textual_data {
			width: 50%;
		}

		.data_triple_grid > div > h3 {
			font-size: 26px;
		}

		.data_triple_grid {
			grid-row-gap: 30px !important;
		}
	}

	:global(.card_container.hidden) {
		transform: translate(-50%, 200%) scale(0.8);
		transition:
			transform 1.5s cubic-bezier(0.165, 0.84, 0.44, 1),
			opacity 0.4s cubic-bezier(0.165, 0.84, 0.44, 1);
		transition-delay: 0.2s;
		will-change: transform, opacity;
	}

	.card_columns {
		width: 50%;
		height: 100%;
		border-radius: 20px;
		background-color: rgb(255, 255, 255);
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		align-items: center;
		border-radius: 20px;
		overflow: hidden;
	}

	.card_columns:nth-of-type(1) {
		align-items: center;
		justify-content: center;
		position: relative;
	}

	.card_columns:nth-of-type(2) {
		background-color: transparent;
		width: max-content;
		padding: 10px;
	}

	.card_columns:nth-of-type(2) > div {
		display: flex;
		flex-direction: column;
		row-gap: 35px;
		width: 100%;
	}

	.data_header {
		width: 100%;
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: flex-start;
		overflow: visible;
	}

	.data_header > :nth-child(1) {
		display: flex;
		flex-direction: row;
		align-items: flex-start;
		column-gap: 10px;
		overflow: visible;
	}

	.data_header > div > h2 {
		font-size: 48px;
	}

	.data_header > div > p {
		color: #8e8e8e;
	}

	.data_grid {
		width: 100%;
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		align-items: left;
		row-gap: 70px;
	}

	.data_grid > div {
		display: flex;
		flex-direction: column;
		row-gap: 0px;
		width: 100%;
	}

	.data_triple_grid {
		width: 100%;
		display: grid;
		grid-template-columns: 1fr 1fr 1fr;
		grid-row-gap: 10px;
		grid-column-gap: 20px;
		align-items: flex-start;
	}

	.data_triple_grid > div > h3 {
		text-align: left;
	}

	.number_container {
		width: 180px;
		height: fit-content;
		padding-top: 2px;
		padding-bottom: 2px;
		row-gap: 0px;
		display: flex;
		flex-direction: column;
		overflow: visible;
		align-items: center;
		justify-content: center;
	}

	.single_chart_container > .number_container {
		align-items: flex-end;
		width: 30%;
		min-width: 30%;
	}

	.data_triple_grid > .number_container {
		align-items: flex-start;
	}

	.number_container > h3 {
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		overflow: visible;
		position: relative;
	}

	.number_container > p > span {
		color: #8e8e8e;
	}

	.data_title {
		font-size: 12px;
		color: #3d3d3d;
		width: 100%;
		overflow: ellipsis;
		white-space: nowrap;
	}

	.chart_grandpa {
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		row-gap: 15px;
		align-items: center;
	}

	.chart_grandpa > .data_header {
		color: rgb(0, 0, 0);
	}

	.chart_header {
		display: flex;
		height: fit-content;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
		background-color: white;
		color: black;
		width: 100%;
		border-radius: 20px;
		padding-left: 20px;
		padding-right: 20px;
		border: 1px solid #000000;
	}

	.chart_header > .number_container {
		width: max-content;
		padding-top: 10px;
		padding-bottom: 10px;
		align-items: flex-start;
		justify-content: center;
	}

	.chart_header > .number_container > p {
		color: black;
	}

	.chart_header > .number_container > h3,
	.single_chart_container > :nth-child(1) > h3 {
		display: flex;
		flex-direction: row;
		align-items: center;
		column-gap: 5px;
	}

	.chart_header > .number_container > p,
	.single_chart_container > :nth-child(1) > p {
		color: black;
	}

	.double_column_chart {
		width: 100%;
		min-width: 100%;
		height: 100%;
		background-color: transparent;
		border-radius: 20px;
		padding: 0px;
		column-gap: 20px;
		position: relative;
		min-width: 0;

		display: grid;
		grid-template-columns: repeat(2, 1fr);
		grid-column-gap: 20px;
	}

	.single_column_chart {
		width: 100%;
		height: 100%;
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		align-items: flex-start;
		background-color: transparent;
		row-gap: 5px;
	}

	.column_header {
		width: 100%;
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
	}

	.column_header > h2 {
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		overflow: visible;
		position: relative;
	}

	.single_column_chart:nth-of-type(1) {
		grid-column: 1;
	}

	.single_column_chart:nth-of-type(2) {
		grid-column: 2;
	}

	.single_column_chart > :nth-child(2) {
		height: 100%;
		width: 100%;
		background-color: white;
		border-radius: 20px;
		display: flex;
		justify-content: space-between;
		position: relative;
	}

	.sx_column {
		flex-direction: row;
		justify-content: center;
		column-gap: auto;
		border: 1px solid #000000;
		padding: 20px 20px !important;
	}

	.dx_column {
		flex-direction: column;
		align-items: center;
		border: 1px solid #000000;
		padding: 20px 40px !important;
	}

	.single_chart_container {
		display: flex;
		flex-direction: row;
		width: 100%;
		align-items: flex-start;
		column-gap: 10px;
	}

	.single_chart_container > :nth-child(1) {
		width: 35%;
		height: 100%;
		text-align: right;
	}

	.single_chart_container > :nth-child(1) > h3 {
		flex-direction: row-reverse;
		display: inline-flex;
	}

	.vertical_chart_container {
		display: flex;
		flex-direction: column;
		align-items: flex-end;
		height: 100%;
		justify-content: flex-end;
	}

	:global(.vertical_chart_container > .chart) {
		width: 100%;
		height: 100%;
		flex-direction: row;
		align-items: center;
		justify-content: center;
		column-gap: 10px;
	}

	:global(.vertical_chart_container > .chart > .box_chart) {
		width: 30px;
		height: 100%;
		display: flex;
		flex-direction: column-reverse;
		column-gap: 5px;
		align-items: center;
	}

	:global(.vertical_chart_container > .number_container > p, h3) {
		color: black;
		text-align: center;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
		overflow: visible;
		position: relative;
	}

	:global(.vertical_chart_container > .chart > .box_chart:nth-of-type(1) > div) {
		width: 100%;
		background-color: #7a7a7a;
		border-radius: 5px;
		transition: height 0.8s ease-in-out;
	}

	:global(.vertical_chart_container > .chart > .box_chart:nth-of-type(2) > div) {
		border-radius: 5px;
		width: 100%;
		background-color: #d9d9d9;
		transition: height 0.8s ease-in-out;
	}

	.small_card_container {
		width: max-content;
		height: 115px;
		display: inline-flex;
		flex-direction: row;
		column-gap: 4px;
		padding: 4px;
		justify-content: space-between;
		background-color: rgba(0, 0, 0, 1);
		border-radius: 6px;
		position: static;
		font-size: 8px;
		backface-visibility: hidden;
		-moz-backface-visibility: hidden;
		-webkit-backface-visibility: hidden;
		-ms-backface-visibility: hidden;
		user-select: none;
		opacity: 0;
		transform: scale(0);
		transition:
			transform 0.4s cubic-bezier(0.165, 0.84, 0.44, 1),
			opacity 0.4s cubic-bezier(0.165, 0.84, 0.44, 1);
		transition-delay: 3s;
		cursor: pointer;
		user-select: none;
	}

	.small_card_sphere {
		width: 20px;
		height: 20px;
		border-radius: 100%;
		background-color: yellow;
		position: absolute;
		top: -9%;
		left: -9%;
		opacity: 0;
		transform: scale(0);
		transition:
			transform 0.4s cubic-bezier(0.165, 0.84, 0.44, 1),
			opacity 0.4s cubic-bezier(0.165, 0.84, 0.44, 1);
	}

	:global(.small_card_container.minimized) {
		transform: scale(0);
		transition: transform 0.3s ease-in-out;
	}

	.small_map {
		width: 50%;
		height: 100%;
		background-color: white;
		border-radius: 3px;
		overflow: hidden;
		box-sizing: border-box;
	}

	:global(.small_map > img, picture) {
		width: 100%;
		height: 100%;
		max-width: 100%;
		max-height: 100%;
		object-fit: contain;
	}

	:global(.small_map > picture > img) {
		width: 100%;
		height: 100%;
		object-fit: contain;
		max-width: 100%;
		max-height: 100%;
	}

	.small_info_container {
		width: 100%;
		height: 100%;
		background-color: transparent;
		border-radius: 5px;
		justify-content: space-between;
		display: flex;
		flex-direction: column;
	}

	.small_header {
		display: inline-flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
		width: 100%;
	}

	.small_counters {
		height: fit-content;
		display: flex;
		flex-direction: column;
		row-gap: 1px;
	}

	.small_counters > p {
		font-size: 12px;
		color: white;
	}

	.small_header h2 {
		font-size: 18px;
		overflow: hidden;
		text-overflow: ellipsis;
		white-space: nowrap;
		color: white;
	}

	.small_header p {
		font-size: 12px;
	}

	.worst_best_button {
		width: fit-content;
		height: fit-content;
		border: 1px solid rgb(0, 0, 0);
		background-color: yellow;
		color: black;
		border-radius: 10px;
		padding-right: 10px;
		padding-left: 10px;
	}

	.worst_best_button > p {
		color: black;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}

	.worst_best_button:hover {
		background-color: rgb(168, 168, 0);
		transform: scale(1.05);
		transition: transform 0.1s ease-in-out;
		color: black;
	}

	.worst_best_button:active {
		background-color: rgb(255, 255, 79);
		color: black;
		transform: scale(0.95);
		transition: transform 0.1s ease-in-out;
	}

	.worst_best_button:active > p {
		color: black;
	}

	:global(.markers.show > .small_card_sphere) {
		transition-delay: 0s;
		transform: scale(5);
		opacity: 1;
		transition:
			transform 0.4s cubic-bezier(0.165, 0.84, 0.44, 1),
			opacity 0.4s cubic-bezier(0.165, 0.84, 0.44, 1);
		transition-delay: 0s;
	}

	:global(.markers.show > .small_card_container) {
		transition-delay: 3s;
		transform: scale(5);
		transform-origin: left top;
		opacity: 1;
		transition:
			transform 1s cubic-bezier(0.165, 0.84, 0.44, 1),
			opacity 0.4s cubic-bezier(0.165, 0.84, 0.44, 1);
		transition-delay: 1.5s;
	}

	.sharer_background {
		width: 100%;
		height: 100%;
		position: absolute;
		top: 50%;
		left: 50%;
		z-index: 100;
		opacity: 0.5;
		transform: translate(-50%, -50%);
		background-color: rgba(0, 0, 0, 0.3);
	}

	.sharer_global_container {
		display: flex;
		flex-direction: column;
		row-gap: 10px;
		border: 1px solid black;
		border-radius: 10px;
		padding: 10px;
		width: 350px;
		height: fit-content;
		position: absolute;
		top: 40%;
		left: 50%;
		transform: translate(-50%, -40%);
		z-index: 1000;
		color: black;
		fill: black;

		background-color: white;
		background-size: cover;
		background-position: center;
		background-repeat: no-repeat;
		background-attachment: fixed;
		transition: height 0.8s ease-in-out;
	}

	.sharer_logo_container {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
		height: fit-content;
		column-gap: 10px;
	}

	.sharer_logo_container > button {
		height: 60px;
		width: 70px;
		aspect-ratio: 1/1;
		border-radius: 10%;
		border: 1px solid black;
		background-color: transparent;
		display: flex;
		align-items: center;
		justify-content: center;
		position: static;
		background-color: white;
	}

	.sharer_logo_container > button:hover {
		background-color: rgb(207, 207, 207);
	}

	.sharer_logo_container > button:active {
		transform: scale(0.95);
		transition: transform 0.1s ease-in-out;
		background-color: rgb(0, 0, 0);
		fill: white;
	}

	.sharer_logo_container > button > svg {
		width: 15px;
		height: 15px;
	}

	.sharer_header {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
		height: 40px;
		width: 100%;
		transition: height 0.8s ease-in-out;
	}

	.sharer_header > button {
		background-color: transparent;
		border: none;
		cursor: pointer;
		position: static;
		transition: transform 0.1s ease-in-out;
		padding: 0px;
	}

	.sharer_header > button:hover {
		cursor: pointer;
	}

	.sharer_header > button:active {
		transform: scale(0.9);
		transition: transform 0.1s ease-in-out;
	}

	.sharer_header > button > svg {
		cursor: pointer;
		width: 35px;
		height: 35px;
		fill: black;
	}

	.sharer_header > button > svg:hover {
		transform: scale(1.1);
		transition: transform 0.1s ease-in-out;
	}

	.sharer_link_container {
		display: flex;
		flex-direction: row;
		justify-content: space-between;
		align-items: center;
		height: fit-content;
		width: 100%;
		column-gap: 10px;
	}

	.sharer_link_text {
		width: 100%;
		height: fit-content;
		border: 1px solid black;
		border-radius: 10px;
		height: 40px;
		color: black;
		background-color: white;
	}

	.sharer_link_text > p {
		width: 100%;
		height: 100%;
		text-align: center;
		display: flex;
		align-items: center;
		padding: 10px;
		font-size: 12px;
		user-select: text;
	}

	.sharer_link_text > p::selection {
		background-color: rgba(0, 0, 0, 1);
		color: white;
	}

	.sharer_copier {
		width: 40px;
		height: 40px;
		border: 1px solid black;
		border-radius: 10px;
		position: relative;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 5px;
		fill: black;
	}

	.sharer_copier:hover {
		cursor: pointer;
		background-color: rgb(39, 39, 39);
	}

	.sharer_copier:active {
		transform: scale(0.95);
		transition: transform 0.1s ease-in-out;
	}

	.copier_alert {
		position: absolute;
		left: 50%;
		bottom: -80%;
		transform: translateX(-50%) scale(0.4);
		opacity: 0;
		pointer-events: auto;
		transition:
			opacity 0.8s cubic-bezier(0.4, 0, 0.2, 1),
			transform 0.8s cubic-bezier(0.4, 0, 0.2, 1);
		display: flex;
		align-items: flex-start;
		justify-content: center;
		padding: 5px 10px;
		border-radius: 10px;
		border: 1px solid black;
		width: max-content;
		background-color: white;
	}

	:global(.copier_alert.show) {
		opacity: 1;
		pointer-events: none;
		transform: translateX(-50%) scale(1);
		transition:
			opacity 0.2s cubic-bezier(0.4, 0, 0.2, 1),
			transform 0.2s cubic-bezier(0.4, 0, 0.2, 1);
	}

	.copier_alert > p {
		color: black;
		font-size: 11px;
		text-align: center;
		margin: 0;
	}

	.sharer_copier > svg {
		width: 22px;
		height: 22px;
		z-index: 20;
		fill: black;
	}

	.sharer_copier:active > svg,
	.sharer_copier:hover > svg {
		transform: scale(0.95);
		transition: transform 0.1s ease-in-out;
		fill: white;
	}
</style>
