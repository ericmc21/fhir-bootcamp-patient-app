<script lang="ts">
	import axios from 'axios';
	import { onMount } from 'svelte';
	import { FHIR_BASE_URL } from './config';
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

<div class="max-w-lg rounded-lg border border-blue-100 bg-blue-50 p-6 shadow-sm">
	{#await getPatientDetails()}
		<p class="text-sm text-gray-500">Loading patient details...</p>
	{:then patient}
		<h2 class="mb-2 text-2xl font-semibold text-gray-800">
			Hello, {patient?.name?.[0]?.given?.[0]}
		</h2>
		<p class="text-gray-600">Welcome to your patient record!</p>

		<div class="mt-5 space-y-4">
			<div class="border-b border-gray-200 pb-2">
				<p class="font-medium text-gray-700">Full Name</p>
				<p class="text-gray-800">{patient?.name?.[0]?.text}</p>
			</div>
			<div class="border-b border-gray-200 pb-2">
				<p class="font-medium text-gray-700">Epic Identifier</p>
				<p class="text-gray-800">
					{patient?.identifier?.find(
						(i) => i.system == 'urn:oid:1.2.840.114350.1.13.0.1.7.5.737384.0'
					)?.value || 'N/A'}
				</p>
			</div>
			<div class="border-b border-gray-200 pb-2">
				<p class="font-medium text-gray-700">Date of Birth</p>
				<p class="text-gray-800">{patient?.birthDate}</p>
			</div>
			<div>
				<p class="font-medium text-gray-700">Gender</p>
				<p class="text-gray-800 capitalize">{patient?.gender}</p>
			</div>
		</div>
	{/await}
</div>
