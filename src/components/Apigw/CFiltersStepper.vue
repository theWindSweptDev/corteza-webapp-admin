<template>
  <b-card
    class="apigw shadow-sm mt-3"
    header-bg-variant="white"
    footer-bg-variant="white"
  >
    <c-filter-modal
      :visible="!!selectedFilter"
      :filter="selectedFilter"
      @submit="onSubmit"
      @reset="onReset"
    />

    <b-form
      @submit.prevent="$emit('submit', route)"
    >
      <b-tabs
        active-nav-item-class="active-tab border-0 text-primary bg-white"
        class="border-0 text-primary font-weight-bold"
        content-class="border-bottom"
      >
        <b-tab
          v-for="(step, index) in steps"
          :key="index"
          class="border-0"
          :title="$t(`filters.step_title.${step}`)"
          @click="onActivateTab(index)"
        >
          <b-row class="d-flex flex-column w-100 m-0">
            <c-filters-dropdown
              class="px-1 py-2"
              :available-filters="getAvailableFiltersByStep"
              :filters="getSelectedFiltersByStep"
              @addFilter="onAddFilter"
            />

            <c-filters-table
              ref="filterTable"
              :filters="getSelectedFiltersByStep"
              :selected-row="step.selectedRow"
              :step="index"
              @filterSelect="onFilterSelect"
              @removeFilter="onRemoveFilter"
              @sortFilters="onSortFilters"
            />
          </b-row>
        </b-tab>
      </b-tabs>
    </b-form>
    <template #header>
      <h3 class="m-0">
        {{ $t('filters.title') }}
      </h3>
    </template>
    <c-submit-button
      class="float-right mt-3"
      :processing="processing"
      :success="success"
      :disabled="disabled"
      @submit="$emit('submit')"
    />
  </b-card>
</template>
<script>
import CFilterModal from 'corteza-webapp-admin/src/components/Apigw/CFilterModal'
import CSubmitButton from 'corteza-webapp-admin/src/components/CSubmitButton'
import CFiltersTable from 'corteza-webapp-admin/src/components/Apigw/CFiltersTable'
import CFiltersDropdown from 'corteza-webapp-admin/src/components/Apigw/CFiltersDropdown'

const mapKindToStep = {
  prefilter: 0,
  processer: 1,
  postfilter: 2,
}

export default {
  components: {
    CFilterModal,
    CSubmitButton,
    CFiltersTable,
    CFiltersDropdown,
  },
  props: {
    processing: {
      type: Boolean,
      value: false,
    },
    success: {
      type: Boolean,
      value: false,
    },
    filters: {
      type: Array,
      required: true,
    },
    availableFilters: {
      type: Array,
      required: true,
    },
    steps: {
      type: Array,
      required: true,
    },
  },

  data () {
    return {
      selectedFilter: null,
      selectedTab: 0,
    }
  },

  computed: {
    disabled () {
      return !(this.filters.some(({ updated, created, deleted }) => updated || created || deleted))
    },

    getSelectedFilter () {
      return this.selectedFilter ? this.selectedFilter : null
    },

    getAvailableFiltersByStep () {
      return (this.availableFilters || []).filter(({ kind }) => {
        return mapKindToStep[kind] === this.selectedTab
      })
    },

    getSelectedFiltersByStep () {
      return (this.filters || []).filter(({ kind, deleted }) => {
        return mapKindToStep[kind] === this.selectedTab && !deleted
      }).sort((a, b) => a.weight - b.weight)
    },

    disabledRemoveButton () {
      return !this.filters.some(({ options }) => (options || { checked: false }).checked)
    },
  },

  methods: {
    onAddFilter (filter) {
      const i = this.filters.findIndex(({ ref, deleted }) => ref === filter.ref && !deleted)

      if (i < 0) {
        this.selectedFilter = filter
      } else {
        this.selectedFilter = this.filters[i]
      }
    },

    onSubmit (filter) {
      const i = this.filters.findIndex(({ ref, deleted }) => ref === filter.ref && !deleted)

      if (i < 0) {
        filter.weight = this.getSelectedFiltersByStep.length
        this.filters.push(filter)
      } else {
        this.filters.splice(this.filters.findIndex(({ ref, deleted }) => ref === filter.ref && !deleted), 1, filter)
      }

      this.$emit('update:filters', this.filters)
    },

    onReset () {
      this.selectedFilter = undefined
    },

    onSortFilters (sortedFilters) {
      this.filters.forEach(filter => {
        const i = sortedFilters.findIndex(({ ref, deleted }) => ref === filter.ref && !deleted)

        if (i >= 0) {
          filter.weight = sortedFilters.findIndex(({ ref, deleted }) => ref === filter.ref && !deleted)
          filter.updated = true
        }
      })
      this.filters.sort((a, b) => a.weight - b.weight)
    },

    onRemoveFilter (filter) {
      if (filter.filterID) {
        this.filters.splice(this.filters.findIndex(({ filterID, deleted }) => filterID === filter.filterID && !deleted), 1, { ...filter, deleted: true })
      } else {
        this.filters.splice(this.filters.findIndex(({ ref, deleted }) => ref === filter.ref && !deleted), 1)
      }

      this.$emit('update:filters', this.filters)
    },

    onFilterSelect (filter = {}) {
      this.selectedFilter = { ...filter }
    },

    onActivateTab (index) {
      this.selectedTab = index
    },
  },
}
</script>
<style lang="scss" >

.apigw {
  .h-fit-content{
    height: fit-content;
  }
  .active-tab{
    border-bottom: 3px solid $primary !important;
    &:hover{
      border-bottom: 3px solid $primary !important;
      color:$primary !important;
    }
  }
  .nav-link:hover{
    color:$primary;
    border:none;
  }
}
</style>
