<script setup lang="ts">
import { useThrottleFn, useTitle } from '@vueuse/core';
import { useRouteQuery } from '@vueuse/router';
import { computed, Ref, ref, watch } from 'vue';
import { useI18n } from 'vue-i18n';
import usePagination from '../composables/pagination';
import useSort from '../composables/sort';
import { useRouteQueryArray } from '../composables/useVueRouterArray';
import useBulkEdition from '../composables/bulkEdition';
import { UserBook } from '../model/Book';
import { ReadingEventType } from '../model/ReadingEvent';
import dataService from "../services/DataService";
import BookCard from "./BookCard.vue";
import SortFilterBarVue from "./SortFilterBar.vue";

const { t } = useI18n({
      inheritLocale: true,
      useScope: 'global'
    })

useTitle('Jelu | To read')

const books: Ref<Array<UserBook>> = ref([]);

const { total, page, pageAsNumber, perPage, updatePage, getPageIsLoading, updatePageLoading } = usePagination()

const { sortQuery, sortOrder, sortBy, sortOrderUpdated } = useSort('creationDate,desc')

const { showSelect, selectAll, checkedCards, cardChecked, toggleEdit } = useBulkEdition(modalClosed)

const eventTypes: Ref<Array<ReadingEventType>> = useRouteQueryArray('lastEventTypes', [])

const open = ref(false)

const owned: Ref<string|null> = useRouteQuery('owned', "null")
const ownedAsBool = computed(() => {
  if (owned.value?.toLowerCase() === "null") {
    return null
  } else if (owned.value?.toLowerCase() === "true") {
    return true
  } else {
    return false
  }
  }
)

const getToReadIsLoading: Ref<boolean> = ref(false)

const getToRead = async () => {
  getToReadIsLoading.value = true
  try {
    const res = await dataService.findUserBookByCriteria(eventTypes.value, null,
    true, ownedAsBool.value,
    pageAsNumber.value - 1, perPage.value, sortQuery.value)
    total.value = res.totalElements
    books.value = res.content
    if (! res.empty) {
      page.value =  (res.number + 1).toString(10)
    }
    else {
      page.value = "1"
    }
    getToReadIsLoading.value = false
    updatePageLoading(false)
  } catch (error) {
    console.log("failed get books : " + error);
    getToReadIsLoading.value = false
    updatePageLoading(false)
  }
};

// watches set above sometimes called twice
// so getBooks was sometimes called twice at the same instant
const throttledGetToRead = useThrottleFn(() => {
  getToRead()
}, 100, false)

watch([page, eventTypes, sortQuery, owned], (newVal, oldVal) => {
  console.log("all " + newVal + " " + oldVal)
  if (newVal !== oldVal) {
    throttledGetToRead()
  }
})

function modalClosed() {
  console.log("modal closed")
  throttledGetToRead()
}

getToRead()

</script>

<template>
  <sort-filter-bar-vue
    :open="open"
    :order="sortOrder"
    class="sort-filter-bar"
    @update:open="open = $event"
    @update:sort-order="sortOrderUpdated"
  >
    <template #sort-fields>
      <div class="field">
        <label class="label">{{ t('sorting.sort_by') }} : </label>
        <o-radio
          v-model="sortBy"
          native-value="creationDate"
        >
          {{ t('sorting.date_added_to_list') }}
        </o-radio>
      </div>
      <div class="field">
        <o-radio
          v-model="sortBy"
          native-value="lastReadingEventDate"
        >
          {{ t('sorting.last_reading_event_date') }}
        </o-radio>
      </div>
      <div class="field">
        <o-radio
          v-model="sortBy"
          native-value="title"
        >
          {{ t('sorting.title') }}
        </o-radio>
      </div>
      <div class="field">
        <o-radio
          v-model="sortBy"
          native-value="publisher"
        >
          {{ t('sorting.publisher') }}
        </o-radio>
      </div>
      <div class="field">
        <o-radio
          v-model="sortBy"
          native-value="series"
        >
          {{ t('sorting.series') }}
        </o-radio>
      </div>
    </template>
    <template #filters>
      <div class="field flex flex-col capitalize">
        <label class="label">{{ t('reading_events.last_event_type') }} : </label>
        <o-checkbox
          v-model="eventTypes"
          native-value="FINISHED"
        >
          {{ t('reading_events.finished') }}
        </o-checkbox>
        <o-checkbox
          v-model="eventTypes"
          native-value="CURRENTLY_READING"
        >
          {{ t('reading_events.currently_reading') }}
        </o-checkbox>
        <o-checkbox
          v-model="eventTypes"
          native-value="DROPPED"
        >
          {{ t('reading_events.dropped') }}
        </o-checkbox>
      </div>
      <div class="field">
        <label class="label">{{ t('filtering.owned') }} : </label>
        <div class="field">
          <o-radio
            v-model="owned"
            native-value="null"
          >
            {{ t('filtering.unset') }}
          </o-radio>
        </div>
        <div class="field">
          <o-radio
            v-model="owned"
            native-value="false"
          >
            {{ t('labels.false') }}
          </o-radio>
        </div>
        <div class="field">
          <o-radio
            v-model="owned"
            native-value="true"
          >
            {{ t('labels.true') }}
          </o-radio>
        </div>
      </div>
    </template>
  </sort-filter-bar-vue>
  <div class="flex flex-row justify-between">
    <div class="flex flex-row gap-1 order-last sm:order-first">
      <o-button
        variant="success"
        outlined
        @click="open = !open"
      >
        <span class="icon text-lg">
          <i class="mdi mdi-filter-variant" />
        </span>
      </o-button>
      <button
        v-tooltip="t('bulk.toggle')"
        class="btn btn-outline btn-primary"
        @click="showSelect = !showSelect"
      >
        <span class="icon text-lg">
          <i class="mdi mdi-pencil" />
        </span>
      </button>
      <button
        v-if="showSelect"
        v-tooltip="t('bulk.select_all')"
        class="btn btn-outline btn-accent"
        @click="selectAll = !selectAll"
      >
        <span class="icon text-lg">
          <i class="mdi mdi-checkbox-multiple-marked" />
        </span>
      </button>
      <button
        v-if="showSelect && checkedCards.length > 0"
        v-tooltip="t('bulk.edit')"
        class="btn btn-outline btn-info"
        @click="toggleEdit(checkedCards)"
      >
        <span class="icon text-lg">
          <i class="mdi mdi-book-edit" />
        </span>
      </button>
    </div>
    <h2 class="text-3xl typewriter capitalize">
      {{ t('nav.to_read') }} :
    </h2>
    <div />
  </div>
  <div
    v-if="books.length > 0"
    class="is-flex is-flex-wrap-wrap is-justify-content-center grid grid-cols-2 md:grid-cols-4 xl:grid-cols-6 2xl:grid-cols-8 gap-0 my-3"
  >
    <div
      v-for="book in books"
      :key="book.id"
      class="books-grid-item m-2"
    >
      <book-card
        :book="book"
        :force-select="selectAll"
        :show-select="showSelect"
        class="h-full"
        @update:modal-closed="modalClosed"
        @update:checked="cardChecked"
      />
    </div>
  </div>
  <div
    v-else-if="getToReadIsLoading"
    class="flex flex-row justify-center justify-items-center gap-3"
  >
    <o-skeleton
      class="justify-self-center basis-36"
      height="250px"
      :animated="true"
    />
    <o-skeleton
      class="justify-self-center basis-36"
      height="250px"
      :animated="true"
    />
    <o-skeleton
      class="justify-self-center basis-36"
      height="250px"
      :animated="true"
    />
  </div>
  <div v-else>
    <h2 class="text-3xl typewriter">
      {{ t('labels.nothing_to_read') }}
    </h2>
    <span class="icon is-large">
      <i class="mdi mdi-book-open-page-variant-outline mdi-48px" />
    </span>
  </div>
  <o-pagination
    v-if="books.length > 0"
    v-model:current="pageAsNumber"
    :total="total"
    order="centered"
    :per-page="perPage"
    @change="updatePage"
  />
  <o-loading
    v-model:active="getPageIsLoading"
    :full-page="true"
    :can-cancel="true"
  />
</template>

<style scoped>

label {
  margin: 0 0.5em;
  font-weight: bold;
}

/* fields in side bar slots are shifted to the right and alignment is broken */
.field {
  margin-left: -8px;
}


</style>
