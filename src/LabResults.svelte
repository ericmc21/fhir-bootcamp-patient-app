<script lang="ts">
	import axios from 'axios';
	import { FHIR_BASE_URL } from './config';
	import type { Observation, Bundle, BundleEntry, OperationOutcome } from 'fhir/r4';
	import { format, formatRelative } from 'date-fns';

	export let accessToken: string;
	export let patientId: string;

	const getLabResults = async () => {
		const observationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(
			`${FHIR_BASE_URL}/Observation`,
			{
				params: { subject: patientId, category: 'laboratory' },
				headers: {
					Authorization: `Bearer ${accessToken}`
				}
			}
		);
		console.log(observationResponse.data);
		return observationResponse.data;
	};

	const getLabResultsEntries = (bundle: Bundle<Observation | OperationOutcome>) => {
		if (!bundle?.entry) {
			return [];
		}
		return bundle.entry?.filter(
			(entry) => entry.resource?.resourceType === 'Observation'
		) as BundleEntry<Observation>[];
	};
</script>

<div class="mx-auto mt-10 max-w-md">
	{#await getLabResults()}
		loading...
	{:then labResults}
		<h1 class="text-2xl">Lab Results</h1>

		{#each getLabResultsEntries(labResults) as labResult, i}
			<p class="font-medium">
				<span>
					{labResult.resource?.code?.text}
					{#if labResult?.resource?.effectiveDateTime}
						({formatRelative(new Date(labResult?.resource?.effectiveDateTime), new Date())}):
					{/if}
				</span>
				{labResult?.resource?.valueQuantity?.value}
				{labResult?.resource?.valueQuantity?.unit}
			</p>
		{/each}
	{/await}
</div>
