<template>
  <section class="items mb-4">
    <h2>
      <span class="title">{{ $tc('stacItem', items.length ) }}</span>
      <b-badge v-if="!api && items.length > 1" pill variant="secondary ml-2">{{ items.length }}</b-badge>
      <SortButtons v-if="!api" class="ml-4" v-model="sort" />
    </h2>

    <Pagination ref="topPagination" v-if="showPagination" :pagination="pagination" placement="top" @paginate="paginate" />
    <template v-if="allowFilter">
      <b-button v-if="api" v-b-toggle.itemFilter class="mb-4 mt-2" :class="{'ml-3': showPagination}" :pressed="filtersOpen" variant="outline-primary">
        <b-icon-search /> {{ $t('items.filter') }}
      </b-button>
      <b-collapse id="itemFilter" v-model="filtersOpen">
        <ItemFilter
          v-if="filtersOpen" :title="$t('items.filter')" :stac="stac"
          :value="apiFilters" :collectionOnly="true"
          v-bind="filterComponentProps" @input="emitFilter"
        />
      </b-collapse>
    </template>

    <section class="list">
      <Loading v-if="loading" fill top />
      <b-card-group v-if="chunkedItems.length > 0" columns>
        <Item v-for="item in chunkedItems" :item="item" :key="item.href" />
      </b-card-group>
      <b-alert v-else :variant="hasFilters ? 'warning' : 'info'" show>
        <template v-if="hasFilters">{{ $t('search.noItemsFound') }}</template>
        <template v-else>{{ $t('items.noneAvailableForCollection') }}</template>
      </b-alert>
    </section>

    <Pagination v-if="showPagination" :pagination="pagination" @paginate="paginate" />
    <b-button v-else-if="hasMore" @click="showMore" variant="primary" v-b-visible.300="showMore">{{ $t('showMore') }}</b-button>
  </section>
</template>

<script>
import Item from './Item.vue';
import Loading from './Loading.vue';
import Pagination from './Pagination.vue';
import { BCollapse, BIconSearch } from "bootstrap-vue";
import Utils from '../utils';
import STAC from '../models/stac';
import apiCapabilitiesMixinGenerator from './ApiCapabilitiesMixin';
import { mapState } from 'vuex';

export default {
  name: "Items",
  components: {
    BCollapse,
    BIconSearch,
    Item,
    ItemFilter: () => import('./ItemFilter.vue'),
    Loading,
    Pagination,
    SortButtons: () => import('./SortButtons.vue')
  },
  mixins: [
    apiCapabilitiesMixinGenerator(true)
  ],
  props: {
    items: {
      type: Array,
      required: true
    },
    loading: {
      type: Boolean,
      default: false
    },
    stac: {
      type: Object,
      required: true
    },
    api: {
      type: Boolean,
      default: false
    },
    allowFilter: {
      type: Boolean,
      default: true
    },
    apiFilters: {
      type: Object,
      default: () => ({})
    },
    pagination: {
      type: Object,
      default: () => ({})
    },
    chunkSize: {
      type: Number,
      default: 90
    }
  },
  data() {
    return {
      shownItems: this.chunkSize,
      filtersOpen: false,
      sort: 0
    };
  },
  computed: {
    ...mapState(['uiLanguage']),
    hasMore() {
      return this.items.length > this.shownItems;
    },
    hasFilters() {
      return Object.values(this.apiFilters).filter(filter => !(filter === null || Utils.size(filter) === 0)).length > 1; // > 1 as the limit is always present
    },
    chunkedItems() {
      let items = this.items;
      if (this.sort !== 0) {
        items = items.slice(0).sort((a,b) => STAC.getDisplayTitle(a).localeCompare(STAC.getDisplayTitle(b), this.uiLanguage));
        if (this.sort === -1) {
          items = items.reverse();
        }
      }
      if (!this.api && this.items.length > this.chunkSize) {
        return items.slice(0, this.shownItems);
      }
      else {
        return items;
      }
    },
    showPagination() {
      if (this.api) {
        if (this.hasFilters) {
          return true;
        }
        else if (this.items.length > 0) {
          // Check whether any pagination links are available
          return Object.values(this.pagination).some(link => !!link);
        }
      }
      return false;
    }
  },
  methods: {
    emitFilter(value, reset) {
      this.$emit('filterItems', value, reset);
    },
    showMore() {
      this.shownItems += this.chunkSize;
    },
    paginate(link, placement) {
      if (placement === 'bottom' && this.$refs.topPagination) {
        Utils.scrollTo(this.$refs.topPagination.$el);
      }
      this.$emit('paginate', link);
    }
  }
};
</script>

<style lang="scss" scoped>
.items {

  .list {
    position: relative;
  }

  > h2 {
    .title, .badge {
      vertical-align: middle;
    }
  }
}
</style>
