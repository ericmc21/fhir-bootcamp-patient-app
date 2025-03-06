<script lang="ts">
	import axios from 'axios';
	import { onMount } from 'svelte';
	import { FHIR_BASE_URL } from './config';
	import type { MedicationRequest, Bundle, BundleEntry, OperationOutcome } from 'fhir/r4';

	export let accessToken: string;
	export let patientId: string;

	const getMedications = async () => {
		const medicationResponse = await axios.get<Bundle<MedicationRequest | OperationOutcome>>(
			`${FHIR_BASE_URL}/MedicationRequest`,
			{
				params: { subject: patientId },
				headers: {
					Authorization: `Bearer ${accessToken}`
				}
			}
		);
		console.log(medicationResponse.data);
		return medicationResponse.data;
	};

	const getMedicationRequestEntries = (bundle: Bundle<MedicationRequest | OperationOutcome>) => {
		if (!bundle?.entry) {
			return [];
		}
		return bundle.entry?.filter(
			(entry) => entry.resource?.resourceType === 'MedicationRequest'
		) as BundleEntry<MedicationRequest>[];
	};
</script>

<div class="mx-auto mt-10 max-w-md">
	{#await getMedications()}
		loading...
	{:then medicationList}
		<h1 class="text-2xl">Medication List</h1>

		{#each getMedicationRequestEntries(medicationList) as medication, i}
			<p class="font-medium">
				{i + 1}. {medication.resource?.medicationReference?.display}
			</p>
			<div class="ml-4">
				{#if medication?.resource?.dosageInstruction?.[0].patientInstruction}
					<p>
						Dosage: {medication?.resource?.dosageInstruction?.[0]?.patientInstruction}
					</p>
				{/if}
				{#if medication?.resource?.reasonCode?.[0].text}
					<p>
						Reason: {medication?.resource?.reasonCode?.[0]?.text}
					</p>
				{/if}
			</div>
		{/each}
	{/await}
</div>
