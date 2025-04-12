<!-- Items.vue -->
<template>
  <h1 class="text-4xl font-bold mb-12">Items Management</h1>

  <NavBar
    type="Item"
    :add-modal-as="AddItem"
    :query-key="queryOptions.queryKey"
    v-model:search="filters['global'].value"
  />

  <DataTable
    v-model:filters="filters"
    :value="items?.data"
    :loading="isPending"
    :lazy="true"
    :totalRecords="items?.count || 0"
    paginator
    :rows="rowsPerPage"
    :rowsPerPageOptions="[5, 10, 20, 50]"
    stripedRows
    tableStyle="width: 100%"
    @page="onPage"
  >
    <Column field="id" header="ID" style="width: 400px" v-if="!xs">
      <template #body="slotProps">
        <span class="line-clamp-1 break-all">{{ slotProps.data.id }}</span>
      </template>
    </Column>

    <Column field="title" header="Title">
      <template #body="slotProps">
        <span class="line-clamp-2 break-words">{{ slotProps.data.title }}</span>
      </template>
    </Column>

    <Column field="description" header="Description">
      <template #body="slotProps">
        <span
          class="line-clamp-2 break-words"
          :class="{ 'text-gray-400': !slotProps.data.description }"
        >
          {{ slotProps.data.description || "N/A" }}
        </span>
      </template>
    </Column>

    <Column header="Actions" style="width: 100px">
      <template #body="slotProps">
        <ActionsMenu
          entityName="Item"
          :query-key="queryOptions.queryKey"
          :value="slotProps.data"
          :edit-model-as="EditItem"
          :on-delete="deleteItem"
        />
      </template>
    </Column>

    <template #empty> No item found. </template>
    <template #loading>
      <div class="my-2" data-testid="loading-item-data">
        Loading item data. Please wait...
      </div>
    </template>
  </DataTable>
</template>

<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { useRouter, useRoute } from "vue-router";
import { useMutation, useQuery, useQueryClient } from "@tanstack/vue-query";
import { FilterMatchMode } from "@primevue/core/api";
import { useDisplay } from "@/composables/useDisplay";

import AddItem from "@/components/items/AddItem.vue";
import ActionsMenu from "@/components/common/ActionsMenu.vue";
import EditItem from "@/components/items/EditItem.vue";
import NavBar from "@/components/common/NavBar.vue";

import {
  itemsDeleteItemMutation,
  itemsReadItemsOptions,
} from "@/client/@tanstack/vue-query.gen";
import type { ItemPublic } from "@/client";

const queryClient = useQueryClient();
const router = useRouter();
const route = useRoute();
const { xs } = useDisplay();

const filters = ref({
  global: { value: "", matchMode: FilterMatchMode.CONTAINS },
});

const rowsPerPage = ref(10);
const currentPage = ref(1);

// Initialize currentPage from the route
currentPage.value = Number(route.query.page) || 1;

// Create a computed query options that updates when pagination parameters change
const queryOptions = computed(() =>
  itemsReadItemsOptions({
    query: {
      skip: (currentPage.value - 1) * rowsPerPage.value,
      limit: rowsPerPage.value,
      search: filters.value.global.value || undefined,
    },
  }),
);

const {
  data: items,
  isPending,
  refetch,
} = useQuery({
  ...queryOptions.value,
  queryKey: computed(() => queryOptions.value.queryKey),
  placeholderData: (prevData) => prevData,
});

// Update URL when page changes
watch(currentPage, (newPage) => {
  router.push({
    query: {
      ...route.query,
      page: newPage.toString(),
    },
  });
});

// Watch for filter changes to reset pagination and refetch
watch(
  () => filters.value.global.value,
  () => {
    currentPage.value = 1;
    refetch();
  },
);

// Handle pagination events
const onPage = (event) => {
  currentPage.value = event.page + 1; // PrimeVue uses 0-based indexing for pages
  rowsPerPage.value = event.rows;
  refetch();
};

const { mutateAsync: deleteItemMutation } = useMutation({
  ...itemsDeleteItemMutation(),
  onSuccess: (data, variables) => {
    queryClient.setQueryData(queryOptions.value.queryKey, (old) =>
      old
        ? {
            ...old,
            data: old.data.filter((item) => item.id !== variables.path.id),
          }
        : undefined,
    );
    refetch(); // Refetch to update total count
  },
});

const deleteItem = async (item: ItemPublic) => {
  await deleteItemMutation({ path: { id: item.id } });
};
</script>

<style scoped>
/* Custom styles can be added here if needed */
</style>
