<script lang="ts">
	import axios from 'axios';
	import { FHIR_BASE_URL } from './config';
	import type { Observation, Bundle, BundleEntry, OperationOutcome } from 'fhir/r4';
	import { formatRelative } from 'date-fns';

	export let accessToken: string;
	export let patientId: string;
	export let category: string = 'laboratory';
	export let title: string = 'Lab Results';

	const getObservations = async () => {
		const observationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(
			`${FHIR_BASE_URL}/Observation`,
			{
				params: { subject: patientId, category: category, sort: 'date' },
				headers: { Authorization: `Bearer ${accessToken}` }
			}
		);
		return observationResponse.data;
	};

	const getObservationDisplay = (observation: Observation | undefined) => {
		if (!observation) return '';
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
		if (!bundle?.entry) return [];
		const results = bundle.entry?.filter(
			(entry) => entry.resource?.resourceType === 'Observation'
		) as BundleEntry<Observation>[];

		return results.sort((a, b) => {
			const dateA = new Date(a.resource?.effectiveDateTime || 0).getTime();
			const dateB = new Date(b.resource?.effectiveDateTime || 0).getTime();
			return dateB - dateA; // Sort descending (most recent first)
		});
	};
</script>

<div class="max-w-lg rounded-lg border border-blue-100 bg-blue-50 p-6 shadow-sm">
	{#await getObservations()}
		<p class="text-sm text-gray-500">Loading observations...</p>
	{:then observations}
		<h2 class="mb-2 text-2xl font-semibold text-gray-800">{title}</h2>

		<div class="mt-5 space-y-4">
			{#each getObservationEntries(observations) as observation, i}
				<div class="border-b border-gray-200 pb-3">
					<p class="font-medium text-gray-700">{observation.resource?.code?.text}</p>
					<p class="text-gray-800">
						{getObservationDisplay(observation.resource)}
					</p>
					{#if observation?.resource?.effectiveDateTime}
						<p class="text-sm text-gray-500">
							{formatRelative(new Date(observation?.resource?.effectiveDateTime), new Date())}
						</p>
					{/if}
				</div>
			{/each}
		</div>
	{/await}
</div>
