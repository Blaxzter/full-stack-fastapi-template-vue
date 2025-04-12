<!-- Admin.vue -->
<template>
  <h1 class="text-4xl font-bold mb-12">Users Management</h1>

  <NavBar
    type="User"
    :add-modal-as="AddUserModal"
    :query-key="readUsersQuery.queryKey"
    v-model:search="filters['global'].value"
  />

  <DataTable
    v-model:filters="filters"
    :value="users?.data"
    :loading="usersLoading"
    :lazy="true"
    :totalRecords="users?.count || 0"
    paginator
    :rows="rowsPerPage"
    :rowsPerPageOptions="[5, 10, 20, 50]"
    stripedRows
    @page="onPage"
  >
    <Column v-if="!xs" field="full_name" header="Full name">
      <template #body="slotProps">
        <div class="flex items-center gap-3">
          <div
            class="line-clamp-2 break-words"
            :class="{ 'text-gray-400': !slotProps.data.full_name }"
          >
            {{ slotProps.data.full_name || "N/A" }}
          </div>
          <Tag
            v-if="currentUser?.id === slotProps.data.id"
            severity="info"
            value="You"
          />
        </div>
      </template>
    </Column>

    <Column field="email" header="Email">
      <template #body="slotProps">
        <span class="line-clamp-1 break-all">{{ slotProps.data.email }}</span>
      </template>
    </Column>

    <Column field="is_superuser" header="Role">
      <template #body="slotProps">
        <div class="line-clamp-1 break-all">
          {{ slotProps.data.is_superuser ? "Superuser" : "User" }}
        </div>
      </template>
    </Column>

    <Column field="is_active" header="Status" style="width: 10%">
      <template #body="slotProps">
        <div class="flex align-items-center gap-2">
          <span
            class="w-2 h-2 rounded-full"
            :class="slotProps.data.is_active ? 'bg-green-500' : 'bg-red-500'"
          />
          {{ slotProps.data.is_active ? "Active" : "Inactive" }}
        </div>
      </template>
    </Column>

    <Column header="Actions">
      <template #body="slotProps">
        <ActionsMenu
          entityName="User"
          :query-key="readUsersQuery.queryKey"
          :value="slotProps.data"
          :edit-model-as="EditUser"
          :on-delete="deleteUser"
        />
      </template>
    </Column>

    <template #empty> No users found. </template>
    <template #loading>
      <div class="p-4 text-center">Loading user data. Please wait...</div>
    </template>
  </DataTable>
</template>

<script setup lang="ts">
import { computed, ref, watch } from "vue";
import { useRouter, useRoute } from "vue-router";
import { useMutation, useQuery, useQueryClient } from "@tanstack/vue-query";
import { FilterMatchMode } from "@primevue/core/api";
import { useDisplay } from "@/composables/useDisplay";

import AddUserModal from "@/components/admin/AddUserModal.vue";
import ActionsMenu from "@/components/common/ActionsMenu.vue";
import EditUser from "@/components/admin/EditUser.vue";
import NavBar from "@/components/common/NavBar.vue";

import {
  usersDeleteUserMutation,
  usersReadUsersOptions,
} from "@/client/@tanstack/vue-query.gen";
import type { UserPublic } from "@/client";

import { storeToRefs } from "pinia";
import { useAuthStore } from "@/stores/auth";
const { user: currentUser } = storeToRefs(useAuthStore());

const queryClient = useQueryClient();
const router = useRouter();
const { xs } = useDisplay();

const filters = ref({
  global: { value: "", matchMode: FilterMatchMode.CONTAINS },
});

const rowsPerPage = ref(10);
const currentPage = ref(1);
const route = useRoute();

// Initialize currentPage from the route
currentPage.value = Number(route.query.page) || 1;

// Create the query with reactive parameters
const readUsersQuery = computed(() =>
  usersReadUsersOptions({
    query: {
      skip: (currentPage.value - 1) * rowsPerPage.value,
      limit: rowsPerPage.value,
      search: filters.value.global.value || undefined,
    },
  }),
);

// The query now depends on the computed readUsersQuery
const {
  data: users,
  isPending: usersLoading,
  refetch,
} = useQuery({
  ...readUsersQuery.value,
  queryKey: computed(() => readUsersQuery.value.queryKey),
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

const { mutateAsync: deleteUserMutation } = useMutation({
  ...usersDeleteUserMutation(),
  onSuccess: (data, variables) => {
    queryClient.setQueryData(readUsersQuery.value.queryKey, (old) =>
      old
        ? {
            ...old,
            data: old.data.filter((user) => user.id !== variables.path.user_id),
          }
        : undefined,
    );
    refetch(); // Refetch to update total count
  },
});

const deleteUser = async (user: UserPublic) => {
  if (currentUser.value?.id !== user.id) {
    await deleteUserMutation({ path: { user_id: user.id } });
  }
};
</script>
