<template>
  <div class="container-fluid p-0 m-0 fixed">
    <div class="row p-0 m-0">
      <div
        class="col container-fluid sidebar-container d-none d-lg-block sticky-top"
      >
        <category-sidebar :countFor="0" :isLoading="isLoadingTokens" />
      </div>
      <div class="col container-fluid content-container">
        <div class="row ps-y-16 ps-x-16 sticky-top tab-header">
          <div
            class="col-12 col-lg cat-switch d-flex d-lg-none ms-b-16 ms-b-lg-0 justify-content-between justify-content-lg-start"
          >
            <categories-selector :countFor="0" :isLoading="isLoadingTokens" class="category-wrapper" />
          </div>
          <div
            class="col-12 col-lg cat-switch d-none d-lg-flex ms-b-16 ms-b-lg-0 justify-content-between justify-content-lg-start"
          >
            <div
              v-if="!selectedCategory"
              class="category d-flex ps-x-16 ps-y-8 cursor-pointer"
            >
              <img
                :src="allCategory.img_url"
                :alt="allCategory.name"
                class="icon align-self-center ms-r-12"
              />
              <div class="font-body-large align-self-center font-medium">
                {{ allCategory.name }}
              </div>
              <div class="count ps-l-12 font-body-large ml-auto">
                {{ allCategory.count }} {{ $t('collectibles') }}
              </div>
            </div>
            <div
              v-if="selectedCategory"
              class="category d-flex ps-x-16 ps-y-8 cursor-pointer"
            >
              <img
                :src="selectedCategory.img_url"
                :alt="selectedCategory.name"
                class="icon align-self-center ms-r-12"
              />
              <div class="font-body-large align-self-center font-medium">
                {{ selectedCategory.name }}
              </div>
              <div class="count ps-l-12 font-body-large ml-auto">
                {{
                  selectedCategory.count ||
                  (displayedTokens && displayedTokens.length) ||
                  0
                }}
                {{ $t('collectibles') }}
              </div>
            </div>
          </div>
          <div
            class="col-12 col-lg search-sort d-flex justify-content-between justify-content-lg-end"
          >
            <search-box
              class="search-box ms-r-0 ms-r-sm-6"
              placeholder="Search NFT..."
              :change="handleSearchInput"
            />
            <sort-dropdown
              class="dropdown-filter ms-l-0 ms-l-sm-6"
              :sortItems="sortItems"
              :change="onSortSelect"
            />
          </div>
        </div>

        <div
          class="row ps-x-16 ps-y-40 d-flex justify-content-center justify-content-lg-start"
        >
          <no-item
            v-if="orderFullList.length <= 0 && !isLoadingTokens"
            class="ps-b-120"
            :message="emptyMsg"
          />
          <no-item
            v-else-if="searchedTokens.length === 0 && !isLoadingTokens"
            class="ps-b-120"
            :message="$t('searchNotFound')"
          />

          <sell-card
            v-for="order in searchedTokens"
            :key="order.id"
            :order="order"
            :searchInput="searchInput"
          />
        </div>
        <div
          class="row ps-x-16 ps-y-40 d-flex justify-content-center text-center"
        >
          <!-- matic loader here -->
          <button-loader
            v-if="
              (hasNextPage && searchedTokens && searchedTokens.length > 0) ||
              isLoadingTokens
            "
            class="mx-auto"
            :loading="isLoadingTokens"
            :loadingText="$t('loading')"
            :text="$t('loadMore')"
            block
            lg
            color="light"
            :click="loadMore"
          />
        </div>
      </div>
    </div>

    <notification-modal v-if="showNotification" @close="onNotificationClose" />
  </div>
</template>

<script>
import Vue from 'vue'
import Component from 'nuxt-class-component'
import { mapGetters, mapState } from 'vuex'
import { fuzzysearch } from '~/helpers'
import { VueWatch, VueDebounce } from '~/components/decorator'

import SellCard from '~/components/lego/sell-card'
import CategoriesSelector from '~/components/lego/categories-selector'
import SearchBox from '~/components/lego/search-box'
import SortDropdown from '~/components/lego/sort-dropdown'
import NoItem from '~/components/lego/no-item'
import moment from 'moment'
import { LOCAL_STORAGE } from "~/constants";
import { LocalStorage } from "~/utils";

import CategorySidebar from '~/components/lego/account/category-sidebar'
import NotificationModal from '~/components/lego/notification-modal'

@Component({
  props: {},
  components: {
    SellCard,
    CategoriesSelector,
    SearchBox,
    SortDropdown,
    NoItem,
    CategorySidebar,
    NotificationModal,
  },
  computed: {
    ...mapGetters('page', [
      'selectedFilters',
      'selectedCategory',
      'selectedCategoryId',
      'activeSort',
    ]),
    ...mapState('page', ['isCategoryFetching']),
    ...mapState('order', {
      orderFullList: (state) => state.orders,
    }),
    ...mapGetters('category', ['categories', 'allCategory']),
  },
  middleware: [],
  mixins: [],
})
export default class Index extends Vue {
  limit = Vue.appConfig.defaultPageSize
  searchInput = null
  fuzzysearch = fuzzysearch
  emptyMsg = {
    title: 'Oops! No item found.',
    description: 'We didn’t found any item that is on sale.',
    img: true,
  }
  showNotification = false

  sortItems = [
    {
      id: 0,
      name: 'Newest',
      filter: '-created',
    },
    {
      id: 1,
      name: 'Oldest',
      filter: '+created',
    },
    {
      id: 2,
      name: 'Price low to high',
      filter: '+usd_price',
    },
    {
      id: 3,
      name: 'Price high to low',
      filter: '-usd_price',
    },
  ]

  hasNextPage = true
  displayTokens = 0
  isLoadingTokens = true

  showModal = false

  mounted() {
    this.$store.dispatch('page/clearFilters')
    // this.$store.dispatch('token/reloadBalances')
    const timestamp = moment().unix()
    const localStorageTimestamp = LocalStorage.get(LOCAL_STORAGE.notificationAccept)
    if (!localStorageTimestamp ||
      parseInt(localStorageTimestamp) + 3600 < timestamp) 
    {
      this.onNotificationOpen()
    }
  }

  // Wathers
  @VueWatch('selectedFilters', { immediate: true, deep: true })
  @VueDebounce(500)
  async onFilterChanged() {
    if (this.isCategoryFetching) {
      return
    }
    this.hasNextPage = true
    this.$store.commit('page/setIsCategoryFetching', true)
    await this.fetchOrders({ filtering: true })
    this.$store.commit('page/setIsCategoryFetching', false)
  }

  // handlers
  onSortSelect(item) {
    this.$store.commit('page/selectedSort', item.filter)
  }

  onNotificationOpen() {
    this.showNotification = true
    const timestamp = moment().unix()
    LocalStorage.set(LOCAL_STORAGE.notificationAccept, timestamp)
  }

  onNotificationClose() {
    this.showNotification = false
  }

  onModalShow() {
    this.showModal = true
  }

  onModalClose() {
    this.showModal = false
  }

  handleSearchInput(val) {
    const formattedString = val.trim()
    this.$store.commit('page/setSearchString', formattedString)
  }

  // Get
  get displayedTokens() {
    return this.orderFullList || []
  }

  get searchedTokens() {
    return this.orderFullList
  }

  async fetchOrders(options = {}) {

    // Do not remove data while fetching
    if (!this.hasNextPage) {
      return
    }
    this.isLoadingTokens = true
    try {
      let offset = this.orderFullList.length

      if (options && options.filtering) {
        // Start from page one with new filter
        offset = 0
      }

      const payload = {
        offset: offset,
        limit: this.limit,
        category: this.selectedCategoryId,
        sort: this.activeSort ? this.activeSort : this.sortItems[0].filter,
        searchString: (this.selectedFilters.searchString && this.selectedFilters.searchString.length > 0) ? this.selectedFilters.searchString : ''
      }

      const data = await this.$store.dispatch('order/getOrders', payload)
      this.hasNextPage = data.has_next_page
    } catch (error) {
      this.$logger.error(error)
    }
    this.isLoadingTokens = false
  }

  updateCategories() {
    this.$store.dispatch('category/fetchCategories')
  }

  loadMore() {
    this.fetchOrders()
  }
}
</script>

<style lang="scss" scoped>
@import '~assets/css/theme/_theme';

.sticky-top {
  &.tab-header {
    top: $navbar-local-height !important;
    background-color: light-color('700');
  }
  &.sidebar-container {
    top: $navbar-local-height !important;
    background-color: light-color('700');
    overflow-x: hidden;
    overflow-y: scroll;
  }
}
.category {
  background-color: light-color('700');
  box-sizing: border-box;

  .icon {
    width: 24px;
    height: 24px;
  }
  .count {
    color: dark-color('300') !important;
  }
}
.search-box {
  max-width: 264px;
  width: 100%;
}
.dropdown-filter,
.search-box {
  height: 44px;
}

.sidebar-container {
  padding: 12px !important;
  max-width: 348px;
  height: 100%;
  border-right: 1px solid light-color('500');
  height: 90vh;
  border-right: 1px solid #f3f4f7;
  overflow-y: scroll;

  /* Hide scrollbar for Chrome, Safari and Opera */
  &::-webkit-scrollbar {
    display: none;
  }
  -ms-overflow-style: none; /* IE and Edge */
  scrollbar-width: none; /* Firefox */
}

@media (max-width: 768px) {
  .sticky-top {
    &.tab-header {
      position: relative;
      top: 0 !important;
      z-index: inherit;
    }
  }
  .category-wrapper {
    width: 100% !important;
    // margin: 0 !important;
  }
  .search-box {
    max-width: 100%;
    width: 100%;
  }
}

@media (max-width: 520px) {
  .search-sort,
  .cat-switch {
    justify-content: center;
    flex-direction: column;
  }

  .dropdown-filter {
    margin-top: 16px;
  }
}
</style>
