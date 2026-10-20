<template>
  <!-- the linked record takes the whole row of the show table and sits in the card like a group: a semibold title, its fields in a card under it -->
  <td colspan="2" class="af-foreign-inline-show block md:table-cell px-5 py-4">
    <div v-if="loading" role="status" class="max-w-sm animate-pulse">
      <div class="h-2.5 w-48 mb-4 rounded-full bg-lightSkeletonBackgroundColor dark:bg-darkSkeletonBackgroundColor"></div>
      <div class="h-2 max-w-[360px] mb-2.5 rounded-full bg-lightSkeletonBackgroundColor dark:bg-darkSkeletonBackgroundColor"></div>
      <div class="h-2 mb-2.5 rounded-full bg-lightSkeletonBackgroundColor dark:bg-darkSkeletonBackgroundColor"></div>
      <div class="h-2 max-w-[330px] mb-2.5 rounded-full bg-lightSkeletonBackgroundColor dark:bg-darkSkeletonBackgroundColor"></div>
      <div class="h-2 max-w-[300px] mb-2.5 rounded-full bg-lightSkeletonBackgroundColor dark:bg-darkSkeletonBackgroundColor"></div>
      <div class="h-2 max-w-[360px] rounded-full bg-lightSkeletonBackgroundColor dark:bg-darkSkeletonBackgroundColor"></div>
      <span class="sr-only">{{ $t('Loading...') }}</span>
    </div>
    <template v-else>
      <h4 class="af-foreign-inline-show-title mb-3 text-sm font-semibold text-lightShowTableHeadingText dark:text-darkShowTableHeadingText">
        {{ listResource.label }} {{$t('inline record')}}
      </h4>
      <ShowTable
        v-if="parentRecord"
        :columns="filteredColumns"
        :resource="listResource"
        :record="parentRecord"
      />
      <!-- no linked record is an empty value, shown as a faint dash like empty fields -->
      <span v-else class="text-sm text-lightInputPlaceholderText dark:text-darkInputPlaceholderText">-</span>
    </template>
  </td>
</template>


<script setup lang="ts">
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
    return;
  }

  const foreignKeyValue = props.record[parentResourceRefColumn.value.name];
  if (!foreignKeyValue) {
    console.warn('No foreign key value found in the record', props.record);
    parentRecord.value = null;
    loading.value = false;
    return;
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
    return;
  }
  if (data.error) {
    showErrorTost(data.error);
    parentRecord.value = null;
    loading.value = false;
    return;
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

