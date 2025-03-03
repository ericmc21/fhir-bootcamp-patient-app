<script lang="ts">
	import axios from 'axios';
	import { onMount } from 'svelte';
	import { FHIR_BASE_URL } from '../config';
	import type { Patient } from 'fhir/r4';

	export let accessToken: string;
	export let patientId: string;

	const getPatientDetails = async () => {
		const patientResponse = await axios.get<Patient>(`${FHIR_BASE_URL}/Patient/${patientId}`, {
			headers: {
				Authorization: `Bearer ${accessToken}`
			}
		});
		return patientResponse.data;
	};
</script>

<div class="mx-auto mt-10 max-w-md">
	{#await getPatientDetails()}
		loading...
	{:then patient}
		<h1 class="text-2xl">Hello, {patient?.name?.[0]?.given?.[0]}</h1>
		<p>Welcome to your patient record!</p>
		<p class="mt-5">
			<span class="font-semibold">Full name: {patient?.name?.[0]?.text}</span>
		</p>
		<p>
			<span class="font-semibold"
				>Epic identifier: {patient.identifier?.find(
					(i) => i.system == 'urn:oid:1.2.840.114350.1.13.0.1.7.5.737384.0'
				)?.value}</span
			>
		</p>
		<p>
			<span class="font-semibold">Date of birth: {patient?.birthDate}</span>
		</p>
		<p>
			<span class="font-semibold">Gender: <span class="capitalize"> {patient?.gender}</span></span>
		</p>
	{/await}
</div>
