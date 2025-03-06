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
	import PatientDetails from '$lib/PatientDetails.svelte';

	let tokenResponse: {
		access_token: string;
		id_token: string;
		patient: string;
		scope: string;
	};

	let loading = true;

	const getSeconds = (date: Date) => {
		return Math.round(date.getTime() / 1000);
	};

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
		//console.log('TRS: ' + tokenResponseString);
		if (tokenResponseString) {
			// TODO: check if tokenResponse expired
			const tokenResponseTemp = JSON.parse(tokenResponseString);
			const { issued_at_in_secs } = tokenResponseTemp;
			const expires_in_secs = issued_at_in_secs + tokenResponseTemp.expires_in;
			const now_in_secs = getSeconds(new Date());

			if (now_in_secs > expires_in_secs) {
				localStorage.removeItem(TOKEN_RESPONSE_LOCAL_STORAGE_KEY);
			} else {
				//console.log('TRT: ' + JSON.stringify(tokenResponseTemp));
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
</script>

<main>
	{#if loading}
		Loading...
	{:else if tokenResponse}
		<PatientDetails accessToken={tokenResponse.access_token} patientId={tokenResponse.patient} />
	{:else}
		<div class="my-20 flex justify-center">
			<button
				class="bg-black p-3 text-white"
				on:click={() => {
					initiateAuthorizationRequest();
				}}>Sign in with Epic</button
			>
		</div>
	{/if}
</main>
