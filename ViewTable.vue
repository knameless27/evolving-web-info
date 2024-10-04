<script lang="ts" setup>
import { FieldDependencies } from '@/schemas/FormMain'
import { OptionsTable } from '@/schemas/Table'
import { can } from '@layouts/plugins/casl'

/**
 * Props interface defines the properties passed to the component.
 */
interface Props {
  optionsFilter?: any // Filter options for the table
  secondParamList?: number // Filter options for the table
  fields: any // Fields definition for the table
  listFunction: Function // Function to list items
  filterFunction: Function // Function to filter items
  editFunction?: Function // Function to edit an item
  createFunction?: Function // Function to create a new item
  fieldDependencies?: FieldDependencies
  deleteFunction?: Function // Function to delete an item
  loadItem?: Function // Function to load a specific item
  execValidation?: Function // Function to load a specific item
  permission?: string // Permission key for filtering
  titleTable: string // Title of the table
  iconTable: string // Icon for the table
  defaultActions?: boolean // Whether to show default actions in the table
  createBtn?: boolean // Whether to show the create button
  noFilter?: boolean // Whether to show the filter
  createForm?: any // Form configuration for creating items
  editForm?: any // Form configuration for editing items
  isCustomAddAndEdit?: boolean // Whether to use custom add and edit forms
  additionalValidation?: boolean // Whether to use custom add and edit forms
}

// Destructure props using defineProps
const {
  optionsFilter,
  fields,
  listFunction,
  filterFunction,
  noFilter,
  titleTable,
  iconTable,
  defaultActions,
  editFunction,
  deleteFunction,
  createFunction,
  createBtn,
  createForm,
  editForm,
  loadItem,
  secondParamList,
  isCustomAddAndEdit,
  execValidation,
  additionalValidation,
} = defineProps<Props>()

// Emit setup
const emit = defineEmits()

// Internationalization setup
const { t } = useI18n()

// Status options for filtering
const statusOptions = computed(() => [
  { name: t('Active'), value: true },
  { name: t('Inactive'), value: false },
])

// Reactive variables and references
const trueFilter = ref(getFilterOptions(optionsFilter))
const optionsTable = ref<OptionsTable>({
  page: 1,
  itemsPerPage: 10,
  sortBy: [''],
  sortDesc: [false],
  total: 0,
  lastPage: 0,
})
const items = ref<any[]>([])
const seeCrud = ref(false)
const isFiltered = ref(false)
const isEdit = ref(false)
const itemIdInAction = ref(0)
const seeDelete = ref(false)
const loading = ref(false)
const filterParams = ref({
  name: null,
  status: null,
})

getList()

/**
 * Function to get filter options
 * @param filterOptions - Optional filter options
 * @returns Array of filter options
 */
function getFilterOptions(filterOptions?: any) {
  if (filterOptions) return filterOptions
  return [
    {
      label: 'Name',
      fieldType: 'text',
      key: 'name',
    },
    {
      label: 'Status',
      fieldType: 'select',
      key: 'status',
      keySelect: 'value',
      labelSelect: 'name',
      options: statusOptions,
    },
  ]
}

/**
 * Function to get the list of items
 */
async function getList() {
  loading.value = true
  try {
    const rawResponse = await listFunction(
      optionsTable.value.itemsPerPage,
      secondParamList || 1
    )
    const response = returnResponse(rawResponse)
    assignValues(response)
  } catch (error) {
    console.error(error)
  } finally {
    loading.value = false
  }
}

/**
 * Function to handle page change
 * @param event - Page change event
 */
async function pageChanged(event?: any) {
  loading.value = true
  let fetchFunction: any
  if (isFiltered.value) {
    fetchFunction = filterFunction
  } else {
    fetchFunction = listFunction
  }

  const rawResponse = await fetchFunction(
    optionsTable.value.itemsPerPage ?? 10,
    event.page,
    filterParams.value
  )
  const response = returnResponse(rawResponse)
  assignValues(response)
}

/**
 * Function to search with filters
 * @param filterValues - Values for filtering
 */
async function searchFilter(filterValues: any) {
  loading.value = true
  try {
    isFiltered.value = true
    filterParams.value = filterValues
    const rawResponse = await filterFunction(
      optionsTable.value.itemsPerPage ?? 10,
      optionsTable.value.page,
      filterValues
    )
    const response = returnResponse(rawResponse)
    assignValues(response)
  } catch (error) {
    console.error(error)
  } finally {
    loading.value = false
  }
}

/**
 * Function to assign values
 * @params default params of one response with pagination
 */
function assignValues({ data, total, current_page, per_page, last_page }: any) {
  items.value = data
  optionsTable.value.itemsPerPage = per_page
  optionsTable.value.total = total
  optionsTable.value.lastPage = last_page
  optionsTable.value.page = current_page ?? 1
  loading.value = false
}

/**
 * Function to add a new item
 */
async function addItem(): Promise<void> {
  handleCustomAddOrEdit('addCustom', null)
}

/**
 * Function to edit an item
 * @param id - ID of the item to be edited
 */
async function editItem({ id }: { id: number }): Promise<void> {
  handleCustomAddOrEdit('editCustom', id)
}

// The `handleCustomAddOrEdit` function is responsible for managing the logic related to adding or
// editing an item based on the provided `option` and `id` parameters. Here's a breakdown of what it
// does:
const handleCustomAddOrEdit = (option: string, id: number | null): void => {
  if (isCustomAddAndEdit) {
    emit('eventOption', { option, id })
  } else {
    isEdit.value = option === 'editCustom'
    seeCrud.value = true
    itemIdInAction.value = id !== null ? id : 0
  }
}

/**
 * Function to delete an item
 * @param id - ID of the item to be deleted
 */
async function deleteItem({ id }: { id: number }) {
  seeDelete.value = true
  itemIdInAction.value = id
}

/**
 * Function to confirm CRUD operation
 */
async function crudConfirmed() {
  itemIdInAction.value = 0
  isEdit.value = false
  seeCrud.value = false
  seeDelete.value = false
  pageChanged({ page: optionsTable.value.page })
}

/**
 * Function to verify if there is an filter for the view
 */
function verifyFilter() {
  return !noFilter
}

/**
 * Exposing Functions to manage the flow outside of this component itself
 */
defineExpose({
  crudConfirmed,
  getList,
  editItem,
  deleteItem,
})
</script>

<template>
  <div>
    <!-- FilterTable component for filtering items -->
    <FilterTable
      v-if="can('filter', permission) && verifyFilter()"
      :fieldsFilter="trueFilter"
      :loading="loading"
      @createFilter="searchFilter"
      @onChangeOptions="searchFilter"
    />

    <!-- Main table card -->
    <VCard>
      <div class="ma-1">
        <VRow class="mt-2 px-2" justify="end">
          <VCol cols="12" class="pt-0">
            <!-- Title of the table -->
            <TitleTable :title-table="titleTable" :icon-table="iconTable" />
          </VCol>
          <VCol cols="12" class="d-flex justify-end" style="gap: 0.5vw">
            <AppButtonBack class="mr-2" />
            <!-- Create button and additional buttons slot -->
            <VBtn
              color="primary"
              v-if="can('create', permission) && createBtn"
              @click="addItem()"
              >{{ `${t('Add')} ${t(titleTable)}` }}</VBtn
            >
            <slot name="additional-buttons" />
          </VCol>
        </VRow>
      </div>

      <!-- DataTableForm component for displaying items -->
      <DataTableForm
        v-bind="$attrs"
        :loading="loading"
        :options="optionsTable"
        :headers="fields"
        :items="items"
        @onChangeOptions="pageChanged"
      >
        <!-- Slot templates for additional actions -->
        <template v-for="(_, name) in $slots" #[name]="slotProps">
          <slot :name="name" v-bind="slotProps || {}" />
        </template>

        <!-- Default actions for editing and deleting items -->
        <template #item.actions="data">
          <div class="d-flex">
            <!-- crear un slot para accciones personalizadas prev -->
            <slot name="additional-actions-prev" v-bind="data" />
            <template v-if="defaultActions">
              <ActionTable
                v-if="can('edit', permission)"
                icon="tabler-edit"
                label="Edit"
                color="primary"
                @click="editItem(data.item)"
              />
              <ActionTable
                v-if="can('delete', permission)"
                icon="tabler-trash"
                label="Delete"
                color="error"
                @click="deleteItem(data.item)"
              />
            </template>
            <!-- crear un slot para accciones personalizadas next -->
            <slot name="additional-actions-next" v-bind="data" />
          </div>
        </template>
      </DataTableForm>
    </VCard>

    <!-- Form for adding and editing items -->
    <EditAndAddActionForm
      v-if="seeCrud"
      v-model="seeCrud"
      :api="(isEdit ? editFunction : createFunction) || ((e: any) => e)"
      :fields="isEdit ? editForm : createForm"
      :id="itemIdInAction"
      :fieldDependencies="fieldDependencies"
      @actionCancelled=";(seeCrud = false), (itemIdInAction = 0)"
      @actionConfirmed="crudConfirmed"
      :additionalValidation="additionalValidation"
      :tableName="titleTable"
      :execValidation="execValidation"
      :nameFetch="loadItem || ((e: any) => e)"
    />

    <!-- Form for confirming deletion of items -->
    <DeleteActionForm
      v-if="seeDelete"
      :api="deleteFunction || ((e: any) => e)"
      :id="itemIdInAction"
      v-model="seeDelete"
      @deleteConfirmed="crudConfirmed"
      @deleteCancelled="seeDelete = false"
      :tableName="titleTable"
    />
  </div>
</template>
