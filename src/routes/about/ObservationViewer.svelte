<script lang="ts">
  import axios from "axios";
  import { FHIR_BASE_URL } from "./config";
  import {formatRelative, subDays} from 'date-fns'
  import type { Bundle, BundleEntry, MedicationRequest, Observation, OperationOutcome } from "fhir/r4";

  export let accessToken: string
  export let patientId: string
  export let category: string = 'laboratory'
  export let title: string = 'Lab Results'

  const getLabResults = async () => {
    const labObservationResponse = await axios.get<Bundle<Observation | OperationOutcome>>(`${FHIR_BASE_URL}/Observation`, {
      params: {
        subject: patientId,
        category,
        sort: '-date'
      },
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    })
    return labObservationResponse.data
  }

  const getObservationDisplay = (observation: Observation | undefined) => {
    if (!observation) {
      return ''
    }
    const isBp = observation.code?.coding?.find(a=>a.code === '55284-4')
    if (isBp) {
      const systolicComponent = observation.component?.find(a=>a.code?.coding?.find(b=>b.code==='8480-6'))
      const systolic = systolicComponent?.valueQuantity?.value
      const diastolicComponent = observation.component?.find(a=>a.code?.coding?.find(b=>b.code==='8462-4'))
      const diastolic = diastolicComponent?.valueQuantity?.value
      return `${systolic}/${diastolic}`
    }
    if (!observation?.valueQuantity?.unit) {
      return observation?.valueQuantity?.value
    }
    return `${observation?.valueQuantity?.value} ${observation?.valueQuantity?.unit}`

  }

  const getObservationEntries = (bundle: Bundle<Observation | OperationOutcome>): BundleEntry<Observation>[] => {
    if (!bundle?.entry) {
      return []
    }
    const results = bundle.entry?.filter((entry) => entry.resource?.resourceType === 'Observation') as BundleEntry<Observation>[];
    return results.sort((a,b)=>{
      if (a?.resource?.effectiveDateTime && b?.resource?.effectiveDateTime) {
        return new Date(b?.resource?.effectiveDateTime).getTime() - new Date(a?.resource?.effectiveDateTime).getTime();
      }
      return 0
    })
    
  }
</script>
<div class="mt-10 max-w-md mx-auto">
{#await getLabResults()}
  loading...
{:then labResults} 
  <h1 class="text-2xl">
    {title}
  </h1>
  {#each getObservationEntries(labResults) as observation, i}
  <p>
    <span class="font-medium">
      {observation.resource?.code?.text}
      {#if observation?.resource?.effectiveDateTime}
        ({formatRelative(new Date(observation?.resource.effectiveDateTime), new Date())})
      {/if}
      :
    </span>
    {getObservationDisplay(observation.resource)}
  </p>
  <div class="ml-4">
  </div>
  {/each}
{/await}
</div>