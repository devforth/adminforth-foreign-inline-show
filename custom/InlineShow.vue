<template>
  <td colspan="2" class="pb-3 px-0 pt-0">
    <div v-if="loading" role="status" class="max-w-sm animate-pulse">
        <div class="h-2.5 bg-gray-200 rounded-full dark:bg-gray-700 w-48 mb-4"></div>
        <div class="h-2 bg-gray-200 rounded-full dark:bg-gray-700 max-w-[360px] mb-2.5"></div>
        <div class="h-2 bg-gray-200 rounded-full dark:bg-gray-700 mb-2.5"></div>
        <div class="h-2 bg-gray-200 rounded-full dark:bg-gray-700 max-w-[330px] mb-2.5"></div>
        <div class="h-2 bg-gray-200 rounded-full dark:bg-gray-700 max-w-[300px] mb-2.5"></div>
        <div class="h-2 bg-gray-200 rounded-full dark:bg-gray-700 max-w-[360px]"></div>
        <span class="sr-only">{{ $t('Loading...') }}</span>
    </div>
    <div 
      v-else-if="coreStore.record"
      class="relative w-full flex flex-col gap-4"
    >
    <div class="flex items-center gap-1">
      <h4 v-if="parentRecord"
          class="px-6 py-4"
      >{{ listResource.label }} {{$t('inline record')}}</h4>
    </div>  
      <div v-if="parentRecord">
        <ShowTable
          :columns="filteredColumns"
          :resource="listResource"
          :record="parentRecord"
        />
      </div>
    </div>
  </td>
</template>


<script setup lang="ts">
import { useCoreStore } from '@/stores/core';
import { onMounted, ref, computed, watchEffect } from 'vue';
import {callAdminForthApi} from '@/utils';
import { showSuccesTost, showErrorTost } from '@/composables/useFrontendApi';
import ShowTable from '@/components/ShowTable.vue';
import { useI18n } from 'vue-i18n';


const sort = ref([]);
const parentRecord = ref(null);
const props = defineProps(['column', 'record', 'meta', 'resource', 'adminUser']);
const loading = ref(true);
const { t } = useI18n();
const coreStore = useCoreStore();
const listResource = ref(null);

const parentResourceRefColumn = computed(() => {
  if (!props.resource || !listResource.value) return null;

  return props.resource.columns.find(c =>
    c.foreignResource?.polymorphicResources
      ? c.foreignResource.polymorphicResources.some(pr => pr.resourceId === listResource.value.resourceId)
      : c.foreignResource?.resourceId === listResource.value.resourceId
  );
});

onMounted(async () => {
  listResource.value = (await callAdminForthApi({
    path: `/plugin/${props.meta.pluginInstanceId}/get_resource`,
    method: 'POST',
    body: {},
  })).resource;
  await getParentRecord();

});

async function getParentRecord() {
  if (!listResource.value || !parentResourceRefColumn.value) {
    console.warn('No parent resource or reference column found');
    parentRecord.value = null;
    loading.value = false;
  }

  const foreignKeyValue = props.record[parentResourceRefColumn.value.name];
  if (!foreignKeyValue) {
    console.warn('No foreign key value found in the record', props.record);
    parentRecord.value = null;
    loading.value = false;
  }

  const data = await callAdminForthApi({
    path: '/get_resource_data',
    method: 'POST',
    body: {
      source: 'show',
      resourceId: listResource.value.resourceId,
      filters: [{
        field: listResource.value.columns.find(c => c.primaryKey).name,
        operator: 'eq',
        value: foreignKeyValue.pk ?? foreignKeyValue,
      }],
      limit: 1,
      offset: 0,
      sort: Array.isArray(sort.value) ? sort.value : [],
    },
  });
  if (!data) {
    console.error('No data returned from API', data);
    parentRecord.value = null;
    loading.value = false;
  }
  if (data.error) {
    showErrorTost(data.error);
    parentRecord.value = null;
    loading.value = false;
  }
  loading.value = false;
  parentRecord.value = data.data?.[0] || null;
}
const filteredColumns = computed(() => {
  return listResource.value?.columns?.filter(col => {
    const isInjectedField =
      col.virtual && col.name?.startsWith('foreignInlineList_');

    return col.showIn?.show && !isInjectedField;
  }) || [];
});
</script>

