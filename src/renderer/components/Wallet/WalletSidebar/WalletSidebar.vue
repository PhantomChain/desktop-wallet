<template>
  <MenuNavigation
    :id="currentWallet.address"
    ref="MenuNavigation"
    :class="{
      'WalletSidebar--collapsed': !isExpanded,
      'WalletSidebar--expanded': isExpanded
    }"
    class="WalletSidebar justify-start pt-0"
    @input="onSelect"
  >
    <div
      v-if="showMenu"
      class="WalletSidebar__menu flex flex-row px-4 pt-4 justify-around"
    >
      <div
        v-if="!isExpanded"
        v-tooltip="{
          content: $t('WALLET_SIDEBAR.FILTER')
        }"
        class="WalletSidebar__menu__button"
        :class="areFiltersActive ? 'WalletSidebar__menu__button--active' : ''"
        @click="toggleFilters"
      >
        <SvgIcon
          name="filter"
          view-box="0 0 13 20"
        />
      </div>
      <div
        v-if="!isExpanded"
        v-tooltip="{
          content: $t('WALLET_SIDEBAR.EXPAND')
        }"
        class="WalletSidebar__menu__button"
        @click="expand"
      >
        <SvgIcon
          name="expand"
          view-box="0 0 18 14"
        />
      </div>
      <div
        v-if="isExpanded"
        class="WalletSidebar__menu__button"
        @click="toggleFilters"
      >
        <div class="flex items-center">
          <SvgIcon
            name="filter"
            view-box="0 0 13 20"
          />
          <span v-if="isExpanded">
            {{ $t('WALLET_SIDEBAR.FILTER') }}
          </span>
        </div>
      </div>
      <div
        v-if="isExpanded"
        class="WalletSidebar__menu__button"
        @click="collapse"
      >
        <div class="flex items-center">
          <SvgIcon
            name="collapse"
            view-box="0 0 18 14"
          />
          <span v-if="isExpanded">
            {{ $t('WALLET_SIDEBAR.HIDE') }}
          </span>
        </div>
      </div>
    </div>

    <WalletSidebarFilters
      v-if="isFiltersVisible"
      :has-contacts="hasContactsOnly"
      :has-ledger="isLedgerConnected"
      :is-sidebar-expanded="isExpanded"
      :outside-click="true"
      :hide-empty="filters.hideEmpty"
      :hide-ledger="filters.hideLedger"
      :search-query="filters.searchQuery"
      :sort-order="sort.order"
      @filter="applyFilters"
      @sort="applyOrder"
      @close="closeFilters"
    />

    <!-- Placeholder wallet -->
    <MenuNavigationItem
      v-if="isExpanded && !isLoadingLedger && availableWallets.length === 0"
      id="placeholder"
      :is-disabled="true"
      class="WalletSidebar__wallet opacity-37.5 select-none"
    >
      <div
        class="WalletSidebar__wallet__wrapper flex flex-row transition items-center w-full mx-6 py-6 truncate"
      >
        <WalletIdenticonPlaceholder
          :size="50"
          class="WalletSidebar__wallet__identicon flex-no-shrink"
        />
        <div
          class="WalletSidebar__wallet__info flex flex-col font-semibold text-theme-page-text-light overflow-hidden pl-2"
        >
          <span class="block truncate">
            {{ $t('PAGES.DASHBOARD.ADD_WALLET') }}
          </span>
          <span class="font-bold mt-2 text-xl">
            {{ formatter_networkCurrency(0, 2) }}
          </span>
        </div>
        <img
          title="arrow"
          :src="assets_loadImage('arrows/arrow-confirmation.svg')"
          class="WalletIdenticon__placeholder__arrow ml-4"
        >
      </div>
    </MenuNavigationItem>

    <!-- Placeholder Loading Ledger Widget -->
    <MenuNavigationItem
      v-if="isLoadingLedger"
      id="isLoadingLedger"
      :is-disabled="true"
      class="WalletSidebar__wallet__ledger-loader select-none"
    >
      <div
        class="WalletSidebar__wallet__wrapper transition text-sm w-full mx-2 py-6 truncate"
      >
        <Loader />
        <div class="font-semibold text-theme-page-text">
          {{ $t('WALLET_SIDEBAR.LOADING_LEDGER') }}
        </div>
      </div>
    </MenuNavigationItem>

    <!-- List of actual wallets -->
    <TransitionGroup
      :class="{ 'opacity-0': isResizing }"
      class="WalletSidebar__container"
      name="WalletSidebar__container"
      tag="div"
    >
      <!-- This element is necessary for the transition group -->
      <div
        v-for="wallet in selectableWallets"
        :key="wallet.id"
      >
        <MenuNavigationItem
          :id="wallet.id"
          class="WalletSidebar__wallet"
        >
          <div
            slot-scope="{ isActive }"
            :class="{ 'flex flex-row': isExpanded }"
            class="WalletSidebar__wallet__wrapper transition items-center w-full mx-6 py-6 truncate"
          >
            <WalletIdenticon
              :size="50"
              :value="wallet.address"
              class="WalletSidebar__wallet__identicon flex-no-shrink"
            />
            <div
              :class="{
                'text-theme-page-text': isActive,
                'text-theme-page-text-light': !isActive,
                'pt-2': !isExpanded,
                'pl-2': isExpanded
              }"
              class="WalletSidebar__wallet__info flex flex-col font-semibold overflow-hidden"
            >
              <span
                class="flex items-center"
                :class="{ 'justify-center': !isExpanded }"
              >
                <span class="block truncate">
                  {{ wallet_name(wallet.address) || wallet_truncate(wallet.address, !isExpanded ? 6 : (wallet.isLedger ? 12 : 24)) }}
                </span>
                <span
                  v-if="wallet.isLedger"
                  v-tooltip="!isExpanded ? $t('COMMON.LEDGER_WALLET') : ''"
                  :class="{ 'w-5': !isExpanded }"
                  class="ledger-badge"
                >
                  {{ !isExpanded ? $t('COMMON.LEDGER').charAt(0) : $t('COMMON.LEDGER') }}
                </span>
              </span>
              <span
                v-if="isExpanded"
                class="font-bold mt-2 text-xl"
              >
                {{ formatter_networkCurrency(wallet.balance, 2) }}
                <!-- TODO display a +/- n PHANTOM on recent transactions -->
              </span>
            </div>
          </div>
        </MenuNavigationItem>
      </div>
    </TransitionGroup>
  </MenuNavigation>
</template>

<script>
import { clone, filter, isEqual, sortBy, uniqBy } from 'lodash'
import Loader from '@/components/utils/Loader'
import { MenuNavigation, MenuNavigationItem } from '@/components/Menu'
import { WalletIdenticon, WalletIdenticonPlaceholder } from '../'
import SvgIcon from '@/components/SvgIcon'
import WalletSidebarFilters from './WalletSidebarFilters'

export default {
  name: 'WalletSidebar',

  components: {
    Loader,
    MenuNavigation,
    MenuNavigationItem,
    SvgIcon,
    WalletIdenticon,
    WalletIdenticonPlaceholder,
    WalletSidebarFilters
  },

  props: {
    showExpanded: {
      type: Boolean,
      required: false,
      default: false
    },
    showMenu: {
      type: Boolean,
      required: false,
      default: true
    }
  },

  data: vm => ({
    hasBeenExpanded: false,
    isFiltersVisible: false,
    isResizing: false,
    filters: clone(vm.$options.defaultFilters),
    sort: {
      order: 'name-asc'
    }
  }),

  defaultFilters: Object.freeze({
    hideEmpty: false,
    hideLedger: false,
    searchQuery: ''
  }),

  computed: {
    areFiltersActive () {
      return !isEqual(this.filters, this.$options.defaultFilters)
    },

    currentWallet () {
      return this.wallet_fromRoute || {}
    },

    hasContactsOnly () {
      return this.currentWallet && this.currentWallet.isContact && !this.currentWallet.isWatchOnly
    },

    isExpanded () {
      return this.showExpanded || this.hasBeenExpanded
    },

    isLedgerConnected () {
      return this.$store.getters['ledger/isConnected']
    },

    isLoadingLedger () {
      return this.$store.getters['ledger/isLoading'] && !this.ledgerWallets.length
    },

    wallets () {
      return this.hasContactsOnly
        ? this.$store.getters['wallet/contactsByProfileId'](this.session_profile.id)
        : this.$store.getters['wallet/byProfileId'](this.session_profile.id)
    },

    ledgerWallets () {
      return this.$store.getters['ledger/wallets']
    },

    /**
     * Wallets before filtering
     */
    availableWallets () {
      let wallets = this.wallets

      if (this.isLedgerConnected && !this.hasContactsOnly) {
        wallets = uniqBy([
          ...wallets,
          ...this.ledgerWallets
        ], 'address')
      }

      return wallets
    },

    selectableWallets () {
      return this.sortWallets(this.filterWallets(this.availableWallets))
    }
  },

  watch: {
    wallets () {
      this.$refs.MenuNavigation.collectItems()
    }
  },

  async created () {
    this.$eventBus.on('ledger:disconnected', this.ledgerDisconnected)
  },

  beforeDestroy () {
    this.$eventBus.off('ledger:disconnected', this.ledgerDisconnected)
  },

  methods: {
    collapse () {
      this.toggleExpanded(false)
    },

    expand () {
      this.toggleExpanded(true)
    },

    toggleExpanded (toExpand) {
      this.isResizing = true
      setTimeout(() => {
        setTimeout(() => (this.isResizing = false), 125)

        this.hasBeenExpanded = toExpand
        this.$emit(toExpand ? 'expanded' : 'collapsed')
      }, 75)
    },

    onSelect (address) {
      this.$router.push({ name: 'wallet-show', params: { address } })
      this.$emit('select', address)
    },

    closeFilters (context) {
      // To not hide the filters when expanding or collapsing
      const wasToggleExpand = context.path.some(path => {
        if (path.className) {
          return path.className.toString().includes('WalletSidebar__menu__button')
        }
      })

      if (!wasToggleExpand) {
        this.isFiltersVisible = false
      }
    },

    toggleFilters () {
      this.isFiltersVisible = !this.isFiltersVisible
    },

    applyFilters (filters) {
      this.$set(this, 'filters', filters)
    },

    filterWallets (wallets) {
      let filtered = wallets

      if (this.filters.hideLedger) {
        filtered = filter(filtered, wallet => !wallet.isLedger)
      }
      if (this.filters.hideEmpty) {
        filtered = filter(filtered, wallet => wallet.balance > 0)
      }
      if (this.filters.searchQuery) {
        filtered = filter(filtered, ({ address, balance, name }) => {
          let match = [
            address,
            balance.toString()
          ].some(text => text.includes(this.filters.searchQuery))

          if (!match) {
            const alternativeName = this.wallet_name(address)
            const names = alternativeName
              ? [name, alternativeName]
              : [name]

            const query = this.filters.searchQuery.toLowerCase()
            match = names.some(name => name.toLowerCase().includes(query))
          }

          return match
        })
      }

      return filtered
    },

    applyOrder (order) {
      this.$set(this, 'sort', { order })
    },

    sortWallets (wallets) {
      const [attr, order] = this.sort.order.split('-')

      if (attr === 'name') {
        wallets = this.wallet_sortByName(wallets)
      } else if (attr === 'balance') {
        wallets = sortBy(wallets, ['balance', 'name', 'address'])
      } else {
        throw new Error(`Sorting by "${attr}" is not implemented`)
      }

      return order === 'asc' ? wallets : wallets.reverse()
    },

    ledgerDisconnected () {
      const hasCurrentWallet = this.currentWallet && this.currentWallet.address

      if (!hasCurrentWallet || this.currentWallet.isLedger) {
        if (this.$refs.MenuNavigation && this.$route.name === 'wallet-show') {
          if (this.selectableWallets.length) {
            this.$refs.MenuNavigation.switchToId(this.selectableWallets[0].address)
          } else {
            this.$router.push({ name: 'wallets' })
          }
        }
      }
    }
  }
}
</script>

<style lang="postcss">
.WalletSidebar--expanded .WalletSidebar__wallet .MenuNavigationItem__border {
  @apply .hidden
}
</style>

<style lang="postcss" scoped>
.WalletSidebar {
  transition: width 0.1s ease-out;
  @apply .overflow-y-auto
}
.WalletSidebar__menu {
  border-bottom: 0.08rem solid var(--theme-feature-item-alternative);
  @apply .sticky .z-10 .bg-theme-feature .pin-t
}
.WalletSidebar__menu__button {
  @apply .cursor-pointer .fill-current .text-theme-option-button-text .py-2 .my-6;
  transition: color 0.4s;
  align-self: center;
}
.WalletSidebar__menu__button:hover {
  @apply .text-theme-button-text;
}
.WalletSidebar__menu__button span {
  @apply .pl-3 .font-bold;
}
.WalletSidebar__menu__button--active {
  @apply .text-theme-feature-item-indicator;
}
.WalletSidebar__menu__separator__line {
  border-right: 0.08rem solid var(--theme-feature-item-alternative);
  @apply .h-12 .my-4
}

.WalletSidebar--expanded .WalletSidebar__wallet__wrapper {
  @apply .border-b .border-theme-feature-item-alternative .text-left
}
.WalletSidebar--expanded .WalletSidebar__wallet__identicon {
  @apply .mr-2
}

.WalletSidebar--collapsed .WalletSidebar__wallet__wrapper {
  @apply .flex-col .justify-center .border-b .border-theme-feature
}
.WalletSidebar--collapsed .WalletSidebar__wallet__identicon {
  @apply .mb-2
}

.WalletIdenticon__placeholder {
  filter: opacity(20%)
}
.WalletIdenticon__placeholder__arrow {
  width: 40px;
  height: 40px;
  transform: scaleY(-1) scaleX(-1)
}

.WalletSidebar__container {
  transition: opacity 0.5s;
}
.WalletSidebar__container-move {
  transition: transform 0.3s;
}
</style>
