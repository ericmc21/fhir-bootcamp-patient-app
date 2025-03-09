<script lang="ts">
	import { onMount } from 'svelte';
	import {
		CLIENT_ID,
		SMART_AUTH_URL,
		REDIRECT_URI,
		FHIR_BASE_URL,
		SMART_TOKEN_URL,
		CODE_VERIFIER_LOCAL_STORAGE_KEY,
		TOKEN_RESPONSE_LOCAL_STORAGE_KEY
	} from '../config';
	import axios from 'axios';
	import pkceChallenge from 'pkce-challenge';
	import PatientDetails from '../PatientDetails.svelte';
	import MedicationDetails from '../MedicationDetails.svelte';
	import ObservationViewer from '../ObservationViewer.svelte';

	let tokenResponse: {
		access_token: string;
		id_token: string;
		patient: string;
		scope: string;
	};

	let loading = true;
	let selectedSection = 'Patient Details'; // 👈 Track selected section

	const sections = [
		{ name: 'Patient Details', id: 'patient-details' },
		{ name: 'Medications', id: 'medications' },
		{ name: 'Lab Results', id: 'lab-results' },
		{ name: 'Vital Signs', id: 'vital-signs' },
		{ name: 'Back to the top', id: 'patient-details' }
	];

	const getSeconds = (date: Date) => Math.round(date.getTime() / 1000);

	const generateCodeChallenge = async () => {
		const { code_challenge, code_verifier } = await pkceChallenge();
		localStorage.setItem(CODE_VERIFIER_LOCAL_STORAGE_KEY, code_verifier);
		return code_challenge;
	};

	const generateRedirectUrl = (codeChallenge: string) => {
		const authorizationUrl = new URL(SMART_AUTH_URL);
		authorizationUrl.searchParams.set('client_id', CLIENT_ID);
		authorizationUrl.searchParams.set('scope', 'openid fhirUser');
		authorizationUrl.searchParams.set('redirect_uri', REDIRECT_URI);
		authorizationUrl.searchParams.set('response_type', 'code');
		authorizationUrl.searchParams.set('state', '1234567');
		authorizationUrl.searchParams.set('aud', FHIR_BASE_URL);
		authorizationUrl.searchParams.set('code_challenge', codeChallenge);
		authorizationUrl.searchParams.set('code_challenge_method', 'S256');

		return authorizationUrl.href;
	};

	onMount(async () => {
		const code = new URL(window.location.href).searchParams.get('code');
		const codeVerifier = localStorage.getItem(CODE_VERIFIER_LOCAL_STORAGE_KEY);
		const tokenResponseString = localStorage.getItem(TOKEN_RESPONSE_LOCAL_STORAGE_KEY);

		if (tokenResponseString) {
			const tokenResponseTemp = JSON.parse(tokenResponseString);
			const { issued_at_in_secs } = tokenResponseTemp;
			const expires_in_secs = issued_at_in_secs + tokenResponseTemp.expires_in;
			const now_in_secs = getSeconds(new Date());

			if (now_in_secs > expires_in_secs) {
				localStorage.removeItem(TOKEN_RESPONSE_LOCAL_STORAGE_KEY);
			} else {
				tokenResponse = tokenResponseTemp;
			}
		}

		if (!tokenResponse) {
			if (code && codeVerifier) {
				await makeTokenRequest(code, codeVerifier);
				localStorage.removeItem(CODE_VERIFIER_LOCAL_STORAGE_KEY);
			}
		}

		loading = false;
	});

	const initiateAuthorizationRequest = async () => {
		const codeChallenge = await generateCodeChallenge();
		const redirectUrl = generateRedirectUrl(codeChallenge);
		window.location.href = redirectUrl;
	};

	const makeTokenRequest = async (code: string, codeVerifier: string) => {
		const tokenRequestForm = new FormData();
		tokenRequestForm.set('grant_type', 'authorization_code');
		tokenRequestForm.set('code', code);
		tokenRequestForm.set('redirect_uri', REDIRECT_URI);
		tokenRequestForm.set('client_id', CLIENT_ID);
		tokenRequestForm.set('code_verifier', codeVerifier);
		const tokenGeneratedAt = getSeconds(new Date());
		const response = await axios.postForm(SMART_TOKEN_URL, tokenRequestForm);
		tokenResponse = response.data;
		localStorage.setItem(
			TOKEN_RESPONSE_LOCAL_STORAGE_KEY,
			JSON.stringify({ ...tokenResponse, issued_at_in_secs: tokenGeneratedAt })
		);
	};

	// 👇 Scroll to section when menu item is clicked
	const scrollToSection = (sectionId: string) => {
		selectedSection = sections.find((s) => s.id === sectionId)?.name || 'Patient Details';
		document.getElementById(sectionId)?.scrollIntoView({ behavior: 'smooth' });
	};
</script>

<main class="flex min-h-screen bg-gray-100">
	<!-- Sidebar -->
	{#if tokenResponse}
		<aside class="hidden w-64 bg-white p-6 shadow-md md:block">
			<h2 class="border-b pb-2 text-lg font-semibold text-gray-700">Patient Dashboard</h2>
			<ul class="mt-4 space-y-3 text-gray-600">
				{#each sections as section}
					<li>
						<button
							class="w-full rounded-md px-3 py-2 text-left transition-all duration-300
							{selectedSection === section.name
								? 'bg-blue-100 font-semibold text-blue-700'
								: 'hover:text-blue-600'}"
							on:click={() => scrollToSection(section.id)}
						>
							{section.name}
						</button>
					</li>
				{/each}
			</ul>
		</aside>
	{/if}
	<!-- Main Content -->
	<section class="flex-1 p-6">
		{#if loading}
			<div class="flex h-screen items-center justify-center">
				<p class="text-gray-600">Loading patient records...</p>
			</div>
		{:else if tokenResponse}
			<div class="container mx-auto max-w-5xl space-y-6 p-6">
				<!-- Add ID to sections for scrolling -->
				<div
					id="patient-details"
					class="rounded-lg transition-all duration-300
	{selectedSection === 'Patient Details' ? 'bg-blue-50 p-5' : 'bg-white p-4'}"
				>
					<PatientDetails
						accessToken={tokenResponse.access_token}
						patientId={tokenResponse.patient}
					/>
				</div>

				<div
					id="medications"
					class="rounded-lg transition-all duration-300
	{selectedSection === 'Medications' ? 'bg-blue-50 p-5' : 'bg-white p-4'}"
				>
					<MedicationDetails
						accessToken={tokenResponse.access_token}
						patientId={tokenResponse.patient}
					/>
				</div>

				<div
					class="rounded-lg transition-all duration-300
	{selectedSection === 'Observations' ? 'bg-blue-50 p-5' : 'bg-white p-4'}"
				>
					<div id="lab-results">
						<ObservationViewer
							category="laboratory"
							title="Lab Results"
							accessToken={tokenResponse.access_token}
							patientId={tokenResponse.patient}
						/>
					</div>
					<div id="vital-signs">
						<ObservationViewer
							category="vital-signs"
							title="Vital Signs"
							accessToken={tokenResponse.access_token}
							patientId={tokenResponse.patient}
						/>
					</div>
				</div>
			</div>
		{:else}
			<div class="mt-20 flex flex-col items-center space-y-6 text-center">
				<h2 class="text-2xl font-semibold text-gray-800">Welcome to Your Patient Portal</h2>
				<p class="max-w-md text-gray-600">
					Sign in with Epic to securely access your **health records, medications, and lab
					results**. This ensures that your medical data remains **private and protected**.
				</p>
				<button
					class="mt-4 rounded-lg bg-blue-600 px-6 py-3 text-lg font-medium text-white shadow-md transition-all duration-300 hover:bg-blue-700"
					on:click={initiateAuthorizationRequest}
				>
					Secure Sign-In with Epic
				</button>
				<p class="max-w-sm text-sm text-gray-500">
					By signing in, you agree to our <a href="#" class="text-blue-600 hover:underline"
						>privacy policy</a
					>.
				</p>
			</div>
		{/if}
	</section>
</main>
