<template>
  <div class="l-experiments" :class="{ 'no-data': experiments.length === 0 }">
    <l-experiments-empty v-if="experiments.length === 0" @l-add-experiment="onCreateExperiment()" />
    <div v-if="experiments.length > 0" class="l-experiments__header-container">
      <l-page-header
        title="Experiments"
        :description="headerDescription"
        button-label="Create Experiment"
        :column="experiments.length === 0"
        @l-header-action="onCreateExperiment()"
      />
    </div>
    <div v-if="experiments.length > 0" class="l-experiments__table-container">
      <l-experiment-table
        :table-data="experiments"
        @l-experiment-selected="onSelectExperiment($event)"
        @l-workflow-selected="onSelectWorkflow($event)"
        @delete-option-clicked="handleDeleteButtonClicked"
        @view-experiment-results-clicked="onShowExperimentResults($event)"
        @view-workflow-results-clicked="onShowWorkflowResults($event)"
      />
    </div>
    <Teleport to=".sliding-panel">
      <transition name="transition-fade">
        <l-experiment-form v-if="isFormVisible" @l-close-form="onDismissForm" />
      </transition>
      <transition name="transition-fade">
        <LExperimentDetails
          v-if="selectedExperiment"
          :selectedExperiment="selectedExperiment"
          :selectedWorkflow="selectedWorkflow"
          :title="selectedWorkflow ? 'Model Run Details' : 'Experiment Details'"
          @l-experiment-results="onShowExperimentResults($event)"
          @l-job-results="onShowWorkflowResults($event)"
          @l-download-results="onDownloadResults($event)"
          @l-show-logs="onShowLogs"
          @l-close-details="onCloseDetails"
          @delete-button-clicked="handleDeleteButtonClicked"
        />
      </transition>
    </Teleport>
    <l-experiments-drawer
      v-if="showDrawer"
      ref="experimentsDrawer"
      :header="getDrawerHeader()"
      :position="showLogs ? 'bottom' : 'full'"
      @l-drawer-closed="resetDrawerContent()"
    >
      <l-experiment-results
        v-if="showExpResults && selectedExperimentResults && selectedExperimentResults.length"
        :results="selectedExperimentResults"
      />
      <l-experiment-results
        v-if="selectedWorkflowResults && showJobResults"
        :results="selectedWorkflowResults"
      />
      <l-experiment-logs :logs="workflowLogs" v-if="showLogs" />
    </l-experiments-drawer>
  </div>
</template>

<script lang="ts" setup>
import { onMounted, watch, ref, computed, type Ref } from 'vue'
import { storeToRefs } from 'pinia'
import { useExperimentStore } from '@/stores/experimentsStore'
import { useDatasetStore } from '@/stores/datasetsStore'
import { useModelStore } from '@/stores/modelsStore'
import { useSlidePanel } from '@/composables/useSlidePanel'
import LPageHeader from '@/components/layout/LPageHeader.vue'
import LExperimentTable from '@/components/experiments/LExperimentTable.vue'
import LExperimentForm from '@/components/experiments/LExperimentForm.vue'
import LExperimentDetails from '@/components/experiments/LExperimentDetails.vue'
import LExperimentsDrawer from '@/components/experiments/LExperimentsDrawer.vue'
import LExperimentResults from '@/components/experiments/LExperimentResults.vue'
import LExperimentLogs from '@/components/experiments/LExperimentLogs.vue'
import LExperimentsEmpty from '@/components/experiments/LExperimentsEmpty.vue'
import type { Experiment } from '@/types/Experiment'
import { WorkflowStatus, type Workflow } from '@/types/Workflow'
import { workflowsService } from '@/sdk/workflowsService'
import { experimentsService } from '@/sdk/experimentsService'
import { downloadContent } from '@/helpers/downloadContent'
import {
  getExperimentResults,
  transformWorkflowResults,
  type TableDataForExperimentResults,
  type TableDataForWorkflowResults,
} from '@/helpers/getExperimentResults'
import { useConfirm, useToast, type ToastMessageOptions } from 'primevue'

const { showSlidingPanel } = useSlidePanel()
const experimentStore = useExperimentStore()
const datasetStore = useDatasetStore()
const modelStore = useModelStore()
const { selectedDataset } = storeToRefs(datasetStore)
const { models } = storeToRefs(modelStore)
const { experiments } = storeToRefs(experimentStore)

const selectedWorkflowResults: Ref<TableDataForWorkflowResults[]> = ref([])
const selectedExperimentResults: Ref<TableDataForExperimentResults[]> = ref([])

const selectedExperiment = ref<Experiment | undefined>()
const selectedWorkflow = ref<Workflow | undefined>()
const workflowLogs: Ref<string[]> = ref([])
const isPolling = ref(false)
let experimentInterval: number | undefined = undefined

const showDrawer = ref(false)
const experimentsDrawer = ref()
const showLogs = ref()
const showExpResults = ref()
const showJobResults = ref()
const headerDescription = ref(`Experiments are a logical sequence of inference and
evaluation tasks that run sequentially to evaluate an LLM.`)

const isFormVisible = computed(() => showSlidingPanel.value && !selectedExperiment.value)

const getDrawerHeader = () => {
  const isWorkflowResults = selectedWorkflowResults.value && showJobResults.value
  return showLogs.value
    ? 'Logs'
    : isWorkflowResults
      ? `Model Run: ${selectedWorkflow.value?.model}`
      : `Experiment: ${selectedExperiment.value?.name}`
}

const onCreateExperiment = () => {
  showSlidingPanel.value = true
  selectedExperiment.value = undefined
}

const onSelectExperiment = (experiment: Experiment) => {
  selectedExperiment.value = experiments.value.find((e: Experiment) => e.id === experiment.id)
  selectedWorkflow.value = undefined
  showSlidingPanel.value = true
}

const onSelectWorkflow = async (workflow: Workflow) => {
  selectedWorkflow.value = await workflowsService.fetchWorkflowDetails(workflow.id)
  selectedExperiment.value = undefined
  showSlidingPanel.value = true
}

const onShowExperimentResults = async (experiment: Experiment) => {
  selectedExperimentResults.value = await getExperimentResults(experiment, models.value)
  selectedExperiment.value = experiment
  showExpResults.value = true
  showDrawer.value = true
}

const onShowWorkflowResults = async (workflow: Workflow) => {
  const results = await workflowsService.fetchWorkflowResults(workflow)
  if (results) {
    selectedWorkflow.value = workflow
    selectedWorkflowResults.value = transformWorkflowResults(results)
  }
  showDrawer.value = true
  showJobResults.value = true
}

const onDownloadResults = async (workflow: Workflow) => {
  const blob = await workflowsService.downloadResults(workflow.id)
  downloadContent(blob, `${workflow.name}_results.json`)
}

const onShowLogs = () => {
  showLogs.value = true
  showDrawer.value = true
}

const onDismissForm = () => {
  datasetStore.setSelectedDataset(undefined)
  showSlidingPanel.value = false
}

const onCloseDetails = () => {
  showSlidingPanel.value = false
}

const toast = useToast()
const confirm = useConfirm()

async function handleDeleteButtonClicked(selectedItem: Workflow | Experiment) {
  const experimentOrWorkflow = selectedItem || selectedWorkflow.value || selectedExperiment.value
  if (!experimentOrWorkflow) {
    return
  }

  const isWorkflow = experimentOrWorkflow.hasOwnProperty('jobs')

  confirm.require({
    message: isWorkflow
      ? `This will permanently delete this model run from your experiment.`
      : `This will permanently delete the experiment and all its model runs.`,
    header: `Delete  ${isWorkflow ? 'Model Run' : 'Experiment'}?`,
    icon: 'pi pi-info-circle',
    rejectLabel: 'Cancel',
    rejectProps: {
      label: 'Cancel',
      severity: 'secondary',
      // style: 'color: #fff; background-color: #363636; border: none; border-radius: 46px;',
      outlined: true,
    },
    acceptProps: {
      label: isWorkflow ? 'Delete Model Run' : 'Delete Experiment',
      // style: 'color: #fff; background-color: #9F1A1C; border: none; border-radius: 46px; ',
      severity: 'danger',
    },
    accept: async () => {
      if (isWorkflow) {
        const relatedExperiment = experiments.value.find((exp) =>
          exp.workflows.some((w) => w.id === experimentOrWorkflow.id),
        )
        if (relatedExperiment && relatedExperiment.workflows.length === 1) {
          await experimentsService.deleteExperiment(relatedExperiment.id)
        } else {
          await workflowsService.deleteWorkflow(experimentOrWorkflow.id)
        }
        toast.add({
          severity: 'secondary',
          summary: `MODEL RUN DELETED`,
          messageicon: 'pi pi-trash',
          detail: `${experimentOrWorkflow.name}`,
          group: 'br',
          life: 3000,
        } as ToastMessageOptions & { messageicon: string })
      } else {
        await experimentsService.deleteExperiment(experimentOrWorkflow.id)
        toast.add({
          severity: 'secondary',
          summary: `EXPERIMENT DELETED`,
          messageicon: 'pi pi-trash',
          detail: `${experimentOrWorkflow.name}`,
          group: 'br',
          life: 3000,
        } as ToastMessageOptions & { messageicon: string })
      }
      showSlidingPanel.value = false
      selectedExperiment.value = undefined
      selectedWorkflow.value = undefined

      // refetch experiments to remove it from the table
      await experimentStore.fetchAllExperiments()
    },
    reject: () => {},
  })
}

async function retrieveWorkflowLogs() {
  if (selectedWorkflow.value) {
    const logsData = await workflowsService.fetchLogs(selectedWorkflow.value?.id)
    const logs = logsData.logs.split('\n')

    workflowLogs.value = logs
  }
}

function startPollingForWorkflowLogs() {
  workflowLogs.value = []
  if (!isPolling.value) {
    isPolling.value = true
    retrieveWorkflowLogs()
    // Poll every 3 seconds
    experimentInterval = setInterval(retrieveWorkflowLogs, 3000)
  }
}

function stopPollingForWorkflowLogs() {
  if (isPolling.value) {
    isPolling.value = false
    clearInterval(experimentInterval)
    experimentInterval = undefined
  }
}

// function getJobRuntime(jobId: string) {
//   const job = jobs.value.find((job) => job.id === jobId)
//   return job ? job.runTime : undefined
// }

watch(
  selectedExperiment,
  (newValue) => {
    if (newValue?.workflows.some((workflow) => workflow.status === WorkflowStatus.RUNNING)) {
      startPollingForWorkflowLogs()
      return
    } else if (isPolling.value) {
      stopPollingForWorkflowLogs()
    }
  },
  { deep: true },
)

watch(
  selectedWorkflow,
  (newValue) => {
    workflowLogs.value = []
    if (newValue) {
      // switch to the experiment the job belongs
      const found = experiments.value.find((experiment: Experiment) => {
        return experiment.workflows.some((workflow) => workflow.id === newValue.id)
      })
      selectedExperiment.value = found

      retrieveWorkflowLogs()
    }
    if (newValue?.status === WorkflowStatus.RUNNING) {
      startPollingForWorkflowLogs()
      return
    } else if (isPolling.value) {
      stopPollingForWorkflowLogs()
    }
  },
  { deep: true },
)

const resetDrawerContent = () => {
  selectedWorkflowResults.value = []
  showExpResults.value = false
  showJobResults.value = false
  showLogs.value = false
  showDrawer.value = false
}

onMounted(async () => {
  await Promise.all([experimentStore.fetchAllExperiments(), modelStore.fetchModels()])

  if (selectedDataset.value) {
    onCreateExperiment()
  }
})

watch(showSlidingPanel, (newValue) => {
  if (!newValue) {
    selectedExperiment.value = undefined
    selectedWorkflow.value = undefined
  }
})
</script>

<style scoped lang="scss">
@use '@/styles/variables' as *;

.l-experiments {
  $root: &;
  max-width: $l-main-width;
  margin: 0 auto;

  &__header-container {
    margin: auto;
    padding: $l-spacing-1;
    display: grid;
    place-items: center;
    width: 100%;
  }

  &__table-container {
    padding: $l-spacing-1;
    display: grid;
    width: 100%;
  }
  &.no-data {
    max-width: $l-main-width;
    padding: $l-spacing-1;
    display: grid;
    place-content: center;

    .l-experiments__header-container {
      margin-top: 120px;
    }
  }
}
</style>
