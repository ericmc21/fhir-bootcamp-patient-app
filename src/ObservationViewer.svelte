<script lang="ts">
	import axios from 'axios';
	import { FHIR_BASE_URL } from './config';
	import type { Observation, Bundle, BundleEntry, OperationOutcome } from 'fhir/r4';
	import { format, formatRelative } from 'date-fns';

	export let accessToken: string;
	export let patientId: string;
	export let category: string = 'laboratory';
	export let title: string = 'Lab Results';

	const getObservations = async () => {
		const observationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(
			`${FHIR_BASE_URL}/Observation`,
			{
				params: { subject: patientId, category: category, sort: 'date' },
				headers: {
					Authorization: `Bearer ${accessToken}`
				}
			}
		);
		console.log(observationResponse.data);
		return observationResponse.data;
	};

	const getObservationDisplay = (observation: Observation | undefined) => {
		if (!observation) {
			return '';
		}
		const isBp = observation.code?.coding?.find((a) => a.code === '55284-4');
		if (isBp) {
			const systolicComponent = observation.component?.find((a) =>
				a.code?.coding?.find((b) => b.code === '8480-6')
			);
			const systolic = systolicComponent?.valueQuantity?.value;

			const diastolicComponent = observation.component?.find((a) =>
				a.code?.coding?.find((b) => b.code === '8462-4')
			);
			const diastolic = diastolicComponent?.valueQuantity?.value;

			return `${systolic}/${diastolic}`;
		}
		if (!observation?.valueQuantity?.unit) {
			return `${observation?.valueQuantity?.value}`;
		}
		return `${observation?.valueQuantity?.value} ${observation?.valueQuantity?.unit}`;
	};

	const getObservationEntries = (bundle: Bundle<Observation | OperationOutcome>) => {
		if (!bundle?.entry) {
			return [];
		}
		const results = bundle.entry?.filter(
			(entry) => entry.resource?.resourceType === 'Observation'
		) as BundleEntry<Observation>[];

		return results.sort((a, b) => {
			const dateA = new Date(a.resource?.effectiveDateTime || 0).getTime();
			const dateB = new Date(b.resource?.effectiveDateTime || 0).getTime();
			return dateB - dateA; // Descending order
		});
	};
</script>

<div class="mx-auto mt-10 max-w-md">
	{#await getObservations()}
		loading...
	{:then observations}
		<h1 class="text-2xl">{title}</h1>

		{#each getObservationEntries(observations) as observation, i}
			<p class="font-medium">
				<span>
					{observation.resource?.code?.text}
					{#if observation?.resource?.effectiveDateTime}
						({formatRelative(new Date(observation?.resource?.effectiveDateTime), new Date())}):
					{/if}
				</span>
				{getObservationDisplay(observation.resource)}
			</p>
		{/each}
	{/await}
</div>
