<template>
  <div class="bg-accent rounded-b-2xl absolute top-0 left-0 h-48 w-full"></div>
  <div class="dark placeholder-black relative z-10 text-white px-5 pt-8 mb-6">
    <div class="flex items-center mb-4">
      <h1 class="text-xl font-semibold text-white">Automa</h1>
      <div class="flex-grow"></div>
      <ui-button
        v-tooltip.group="t('home.record.title')"
        icon
        class="mr-2"
        @click="state.newRecordingModal = true"
      >
        <v-remixicon name="riRecordCircleLine" />
      </ui-button>
      <ui-button
        v-tooltip.group="
          t(`home.elementSelector.${state.haveAccess ? 'name' : 'noAccess'}`)
        "
        icon
        class="mr-2"
        @click="initElementSelector"
      >
        <v-remixicon name="riFocus3Line" />
      </ui-button>
      <ui-button
        v-tooltip.group="t('common.dashboard')"
        icon
        :title="t('common.dashboard')"
        @click="openDashboard"
      >
        <v-remixicon name="riHome5Line" />
      </ui-button>
    </div>
    <div class="flex">
      <ui-input
        v-model="state.query"
        :placeholder="`${t('common.search')}...`"
        prepend-icon="riSearch2Line"
        class="w-full search-input"
      />
    </div>
  </div>
  <div class="px-5 pb-5 space-y-2">
    <ui-card v-if="Workflow.all().length === 0" class="text-center">
      <img src="@/assets/svg/alien.svg" />
      <p class="font-semibold">{{ t('message.empty') }}</p>
      <ui-button
        variant="accent"
        class="mt-6"
        @click="openDashboard('/workflows')"
      >
        {{ t('home.workflow.new') }}
      </ui-button>
    </ui-card>
    <home-workflow-card
      v-for="workflow in workflows"
      :key="workflow.id"
      :workflow="workflow"
      @details="openDashboard(`/workflows/${$event.id}`)"
      @update="updateWorkflow(workflow.id, $event)"
      @execute="executeWorkflow"
      @rename="renameWorkflow"
      @delete="deleteWorkflow"
    />
  </div>
  <ui-modal v-model="state.newRecordingModal" custom-content>
    <ui-card
      :style="{ height: `${state.cardHeight}px` }"
      class="w-full recording-card overflow-hidden rounded-b-none"
      padding="p-0"
    >
      <div class="flex items-center px-4 pt-4 pb-2">
        <p class="flex-1 font-semibold">
          {{ t('home.record.title') }}
        </p>
        <v-remixicon
          class="text-gray-600 dark:text-gray-300 cursor-pointer"
          name="riCloseLine"
          size="20"
          @click="state.newRecordingModal = false"
        ></v-remixicon>
      </div>
      <home-start-recording
        @record="recordWorkflow"
        @close="state.newRecordingModal = false"
        @update="state.cardHeight = recordingCardHeight[$event] || 255"
      />
    </ui-card>
  </ui-modal>
</template>
<script setup>
import { computed, onMounted, shallowReactive } from 'vue';
import { useI18n } from 'vue-i18n';
import browser from 'webextension-polyfill';
import { useDialog } from '@/composable/dialog';
import { useGroupTooltip } from '@/composable/groupTooltip';
import { sendMessage } from '@/utils/message';
import Workflow from '@/models/workflow';
import HomeWorkflowCard from '@/components/popup/home/HomeWorkflowCard.vue';
import HomeStartRecording from '@/components/popup/home/HomeStartRecording.vue';

const recordingCardHeight = {
  new: 255,
  existing: 480,
};

const { t } = useI18n();
const dialog = useDialog();

useGroupTooltip();

const state = shallowReactive({
  query: '',
  cardHeight: 255,
  haveAccess: true,
  newRecordingModal: false,
});

const workflows = computed(() =>
  Workflow.query()
    .where(({ name }) =>
      name.toLocaleLowerCase().includes(state.query.toLocaleLowerCase())
    )
    .orderBy('createdAt', 'desc')
    .get()
);

function executeWorkflow(workflow) {
  sendMessage('workflow:execute', workflow, 'background');
}
function updateWorkflow(id, data) {
  return Workflow.update({
    where: id,
    data,
  });
}
function renameWorkflow({ id, name }) {
  dialog.prompt({
    title: t('home.workflow.rename'),
    placeholder: t('common.name'),
    okText: t('common.rename'),
    inputValue: name,
    onConfirm: (newName) => {
      updateWorkflow(id, { name: newName });
    },
  });
}
function deleteWorkflow({ id, name }) {
  dialog.confirm({
    title: t('home.workflow.delete'),
    okVariant: 'danger',
    body: t('message.delete', { name }),
    onConfirm: () => {
      Workflow.delete(id);
    },
  });
}
function openDashboard(url) {
  sendMessage('open:dashboard', url, 'background');
}
async function initElementSelector() {
  const [tab] = await browser.tabs.query({ active: true, currentWindow: true });

  try {
    await browser.tabs.sendMessage(tab.id, {
      type: 'automa-element-selector',
    });
  } catch (error) {
    if (error.message.includes('Could not establish connection.')) {
      await browser.tabs.executeScript(tab.id, {
        allFrames: true,
        file: './elementSelector.bundle.js',
      });

      initElementSelector();
    }

    console.error(error);
  }
}
async function recordWorkflow(options = {}) {
  try {
    const flows = [];
    const [activeTab] = await browser.tabs.query({
      active: true,
      currentWindow: true,
    });

    if (activeTab && activeTab.url.startsWith('http')) {
      flows.push({
        id: 'new-tab',
        description: activeTab.url,
        data: { url: activeTab.url },
      });
    }

    await browser.storage.local.set({
      isRecording: true,
      recording: {
        flows,
        name: 'unnamed',
        activeTab: {
          id: activeTab.id,
          url: activeTab.url,
        },
        ...options,
      },
    });
    await browser.browserAction.setBadgeBackgroundColor({ color: '#ef4444' });
    await browser.browserAction.setBadgeText({ text: 'rec' });

    const tabs = await browser.tabs.query({});
    for (const tab of tabs) {
      if (tab.url.startsWith('http')) {
        await browser.tabs.executeScript(tab.id, {
          file: 'recordWorkflow.bundle.js',
        });
      }
    }

    window.close();
  } catch (error) {
    console.error(error);
  }
}

onMounted(async () => {
  const [tab] = await browser.tabs.query({ active: true, currentWindow: true });

  state.haveAccess = /^(https?)/.test(tab.url);
});
</script>
<style>
.recording-card {
  transition: height 300ms cubic-bezier(0.4, 0, 0.2, 1) !important;
}
</style>
