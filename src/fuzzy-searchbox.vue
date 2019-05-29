<template>
  <div class="fuzzy-searchbox">
    <input
      class="fuzzy-searchbox skiptranslate notranslate autocomplete-input"
      aria-haspopup="listbox"
      aria-owns="autocomplete-results"
      :aria-expanded="isOpen"
      role="combobox"
      :placeholder="placeholder"
      type="text"
      spellcheck="false"
      @input="onChange"
      v-model="searchTerm"
      ref="input"
      @keydown.down="onArrowDown"
      @keydown.up="onArrowUp"
      @keydown.enter="onEnter"
      @keydown.escape="onLeave"
      @focus="onClick"
      @blur="onLeave"
      aria-multiline="false"
      aria-autocomplete="list"
      aria-controls="autocomplete-results"
      :aria-label="ariaLabel"
      :aria-activedescendant="activedescendant"
      autocomplete="off"
      autocorrect="off"
      autocapitalize="off"
    />
    <ul
      id="autocomplete-results"
      ref="autocomplete-results"
      v-show="isOpen"
      class="autocomplete-results"
      role="listbox"
    >
      <li
        class="loading"
        v-if="isLoading"
      >
        Loading results...
      </li>
      <li
        v-else
        v-for="(result, i) in results"
        :key="i"
        @click="setResult(result)"
        @mouseover="arrowCounter = i"
        class="autocomplete-result"
        :class="{ 'is-active': isSelected(i) }"
        role="option"
        :id="getId(i)"
        :ref="getId(i)"
        :aria-selected="isSelected(i)"
      >
      {{ result.fuzzySearchboxLabel }}
      </li>
    </ul>
  </div>
</template>
<script>
import Fuse from 'fuse.js'
import throttle from 'lodash/throttle'
import debounce from 'lodash/debounce'
import isEqual from 'lodash/isEqual'

export default{
  props: {
    items: {
      type: Array,
      required: false,
      default: () => []
    },
    isAsync: {
      type: Boolean,
      required: false,
      default: false
    },
    isThrottled: {
      type: Boolean,
      required: false,
      default: false
    },
    isDebounced: {
      type: Boolean,
      required: false,
      default: true
    },
    ariaLabel: {
      type: String,
      required: true
    },
    itemKeys: {
      type: Array,
      required: false
    },
    itemLabels: {
      type: Array,
      required: false
    },
    labelDelimiter: {
      type: String,
      required: false,
      default: ' '
    },
    createLabelFunction: {
      type: Function,
      required: false
    },
    searchByLabelRuleFunction: {
      type: Function,
      required: false
    },
    searchByLabel: {
      type: Boolean,
      required: false,
      default: false
    },
    preSelectedItem: [String, Object],
    placeholder: {
      type: String,
      required: false
    },
    maxItemsDisplayed: {
      type: Number,
      required: false
    },
    showDefaultResults: {
      type: Boolean,
      required: false,
      default: true
    }
  },

  data () {
    return {
      isOpen: false,
      results: [],
      searchTerm: '',
      activedescendant: 'result-item-0',
      isLoading: false,
      arrowCounter: 0,
      selectedItem: {},
      searchOptions: {
        shouldSort: true,
        tokenize: true,
        matchAllTokens: true,
        threshold: 0.5,
        location: 0,
        distance: 100,
        maxPatternLength: 32,
        minMatchCharLength: 1
      }
    }
  },

  mounted () {
    document.addEventListener('click', this.handleClickOutside)
    if (this.searchByLabel) {
      this.searchOptions.keys = this.labelSearchRuleExists() ? ['fuzzySearchboxSearchLabel'] : ['fuzzySearchboxLabel']
    } else {
      this.searchOptions.keys = this.itemKeys
    }
    this.createItemLabels(this.items)
    if (this.preSelectedItem) {
      this.selectedItem = this.preSelectedItem
    }
    this.throttledSearch = throttle(this.filterResults, 600, { 'leading': false })
    this.debouncedSearch = debounce(this.filterResults, 600, { 'leading': false })
  },

  destroyed () {
    document.removeEventListener('click', this.handleClickOutside)
  },

  methods: {
    labelSearchRuleExists: function () {
      return typeof this.searchByLabelRuleFunction === 'function'
    },
    onChange: function () {
      this.isLoading = true
      this.searchTerm = this.$refs['input'].value
      if (!this.isAsync) {
        this.isLoading = true
        this.isThrottled ? this.throttledSearch() : this.isDebounced ? this.debouncedSearch() : this.filterResults()
        this.arrowCounter = 0
        this.isOpen = true
        this.$refs['autocomplete-results'].scrollTop = 0
      }
    },
    onClick: function () {
      if (!this.isOpen) {
        this.isOpen = true
        this.$refs['input'].select()
        if (this.results.length === 0) {
          this.filterResults()
        }
      }
    },
    itemKeysSet: function () {
      if (this.itemKeys) {
        return true
      } else {
        return false
      }
    },
    itemLabelsSet: function () {
      if (this.itemLabels) {
        return true
      } else {
        return false
      }
    },
    filterResults: function () {
      if (!this.searchTerm) {
        if (this.showDefaultResults) {
          this.results = this.maxItemsDisplayed ? this.items.slice(0, this.maxItemsDisplayed) : this.items
          this.isLoading = false
        } else {
          this.isOpen = false
        }
      } else {
        var search = this.searchTerm
        if (!this.searchOptions.matchAllTokens) {
          if (search.substr(-1, 1) !== ' ') {
            search = search + ' '
          }
        } else {
          search = search.trim()
        }
        this.results = this.maxItemsDisplayed ? this.fuse.search(search).slice(0, this.maxItemsDisplayed) : this.fuse.search(search)
        this.isLoading = false
      }
    },
    setResult: function (result) {
      // this.search = this.itemKey ? result[this.itemKey] : result
      this.selectedItem = result
      this.isOpen = false
      this.$emit('input', this.selectedItem)
      this.$emit('change')
    },
    createLabel: function (item) {
      var label = ''
      if (typeof this.createLabelFunction === 'function') {
        label = this.createLabelFunction(item)
        return label
      }
      // Generic case
      var keys = []
      if (this.itemLabelsSet) {
        keys = this.itemLabels
      } else {
        if (this.itemKeysSet) {
          keys = this.itemKeys
        } else {
          keys = ['option'] // The key for arrays should be 'option'. This is yet to be implemented.
        }
      }
      for (var key in keys) {
        label += item[keys[key]] + this.labelDelimiter
      }
      label = label.substring(0, label.lastIndexOf(this.labelDelimiter))
      return label
    },
    onArrowDown: function () {
      if (this.isOpen) {
        if (this.arrowCounter < this.results.length - 1) {
          this.arrowCounter = this.arrowCounter + 1
        }
      }
    },
    onArrowUp: function () {
      if (this.isOpen) {
        if (this.arrowCounter > 0) {
          this.arrowCounter = this.arrowCounter - 1
        }
      }
    },
    onEnter: function () {
      if (this.isOpen) {
        if (this.arrowCounter > -1) {
          this.setResult(this.results[this.arrowCounter])
          this.arrowCounter = 0
        } else {
          this.resetSearchTerm()
        }
        this.isOpen = false
      }
    },
    onLeave: function () {
      setTimeout(() => {
        this.isOpen = false
        this.arrowCounter = 0
        this.resetSearchTerm()
      }, 300)
    },
    handleClickOutside: function (evt) {
      if (!this.$el.contains(evt.target) && this.isOpen) {
        this.isOpen = false
        this.arrowCounter = 0
        this.resetSearchTerm()
      }
    },
    resetSearchTerm: function () {
      if (Object.keys(this.selectedItem).length !== 0) {
        this.searchTerm = this.createLabel(this.selectedItem)
      } else {
        this.searchTerm = ''
      }
    },
    setActiveDescendant: function () {
      this.activedescendant = this.getId(this.arrowCounter)
    },
    getId: function (index) {
      return `result-item-${index}`
    },
    isSelected: function (i) {
      return i === this.arrowCounter
    },
    createItemLabels: function (items) {
      for (var itemCount in items) {
        var item = items[itemCount]
        item.fuzzySearchboxLabel = this.createLabel(item)
        if (this.labelSearchRuleExists()) {
          item.fuzzySearchboxSearchLabel = this.searchByLabelRuleFunction(item.fuzzySearchboxLabel)
        }
      }
      this.isLoading = false
      this.fuse = new Fuse(items, this.searchOptions)
    }
  },

  watch: {
    items: function onItemsChange (val, oldValue) {
      if (!isEqual(val, oldValue)) {
        this.createItemLabels(val)
      }
    },
    itemKeys: function onItemKeysChange (val, oldValue) {
      if (!isEqual(val, oldValue)) {
        if (this.searchByLabel) {
          this.searchOptions.keys = this.labelSearchRuleExists() ? ['fuzzySearchboxSearchLabel'] : ['fuzzySearchboxLabel']
        } else {
          this.searchOptions.keys = this.itemKeys
        }
        this.fuse = new Fuse(this.items, this.searchOptions)
      }
    },
    selectedItem: function onSelectedItemChange (val, oldValue) {
      this.searchTerm = val.fuzzySearchboxLabel
    },
    preSelectedItem: function onPreSelectedItemChange (val, oldValue) {
      if (!isEqual(val, oldValue)) {
        this.selectedItem = JSON.parse(JSON.stringify(this.preSelectedItem))
      }
    },
    arrowCounter: function onArrowCounterChanged (val, oldValue) {
      if (val >= 0) {
        this.setActiveDescendant()
        var target = this.$refs[this.getId(this.arrowCounter)][0]
        var parent = target.parentNode
        var targetVisible = target.offsetTop > parent.scrollTop && target.offsetTop < parent.scrollTop + parent.offsetHeight - target.offsetHeight
        if (!targetVisible) {
          if (val > oldValue) {
            parent.scrollTop = target.offsetTop - parent.offsetHeight + target.offsetHeight
          } else {
            parent.scrollTop = target.offsetTop
          }
        }
      }
    }
  }
}

</script>

<style lang="scss" scoped>
.fuzzy-searchbox {
  position: relative;
  display: inline-block;
  width: inherit;
}

.autocomplete-results {
  padding: 0;
  margin: 0;
  border: 1px solid #eeeeee;
  max-height: 400px;
  overflow-y: auto;
  width: 100%;
  left:0;
  background-color: white;
  position: absolute;
  z-index: 999 !important;
  box-shadow: 0 4px 5px rgba(0,0,0,.15);
}

.autocomplete-result {
  list-style: none;
  text-align: left;
  padding: 4px 2px;
  cursor: pointer;
  display: list-item;
}

.autocomplete-result.is-active {
  background-color: #4aae9b;
  color: white;
}

.autocomplete-input {
  width: inherit;
}
</style>
