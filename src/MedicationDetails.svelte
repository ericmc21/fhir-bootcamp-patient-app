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

<div class="max-w-lg rounded-lg border border-blue-100 bg-blue-50 p-6 shadow-sm">
	{#await getMedications()}
		<p class="text-sm text-gray-500">Loading medications...</p>
	{:then medicationList}
		<h2 class="mb-2 text-2xl font-semibold text-gray-800">Medication List</h2>

		<div class="mt-5 space-y-4">
			{#each getMedicationRequestEntries(medicationList) as medication, i}
				<div class="border-b border-gray-200 pb-4">
					<p class="font-medium text-gray-700">
						{i + 1}. {medication.resource?.medicationReference?.display}
					</p>

					<div class="ml-4 space-y-1 text-gray-800">
						{#if medication?.resource?.dosageInstruction?.[0].patientInstruction}
							<p class="text-sm">
								<span class="font-medium text-gray-700">Dosage:</span>
								{medication?.resource?.dosageInstruction?.[0]?.patientInstruction}
							</p>
						{/if}
						{#if medication?.resource?.reasonCode?.[0].text}
							<p class="text-sm">
								<span class="font-medium text-gray-700">Reason:</span>
								{medication?.resource?.reasonCode?.[0]?.text}
							</p>
						{/if}
					</div>
				</div>
			{/each}
		</div>
	{/await}
</div>
