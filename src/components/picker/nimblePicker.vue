<template>

<div :class="{ 'emoji-mart': true, [`emoji-mart-${this.prefferedTheme}`]: true }" :style="customStyles">
  <div class="emoji-mart-bar" v-if="showCategories">
    <anchors
      :data="mutableData"
      :i18n="mutableI18n"
      :color="color"
      :categories="filteredCategories"
      :active-category="activeCategory"
      @click="onAnchorClick"
    />
  </div>

  <search
    v-if="showSearch"
    ref="search"
    :data="mutableData"
    :i18n="mutableI18n"
    :emojis-to-show-filter="emojisToShowFilter"
    :include="include"
    :exclude="exclude"
    :custom="customEmojis"
    :recent="recentEmojis"
    :auto-focus="autoFocus"
    @search="onSearch"
  />

  <div class="emoji-mart-scroll" :style="{ height: `${height}px`}" ref="scroll" @scroll="onScroll">
    <category
      v-show="searchEmojis"
      :data="mutableData"
      :i18n="mutableI18n"
      id="search"
      name="Search"
      :emojis="searchEmojis"
      :emoji-props="emojiProps"
    />
    <category
      v-for="category in filteredCategories"
      v-show="!searchEmojis && (infiniteScroll || category == activeCategory)"
      ref="categories"
      :key="category.id"
      :data="mutableData"
      :i18n="mutableI18n"
      :id="category.id"
      :name="category.name"
      :emojis="category.emojis"
      :emoji-props="emojiProps"
    />
  </div>
</div>

</template>

<script>

import '../../vendor/raf-polyfill'
import store from '../../utils/store'
import frequently from '../../utils/frequently'
import { deepMerge, measureScrollbar } from '../../utils'
import { uncompress } from '../../utils/data'
import { PickerProps } from '../../utils/shared-props'
import Anchors from '../anchors'
import Category from '../category'
import Search from '../search'

const RECENT_CATEGORY = { id: 'recent', name: 'Recent', emojis: null }
const CUSTOM_CATEGORY = { id: 'custom', name: 'Custom', emojis: [] }

const I18N = {
  search: 'Search',
  notfound: 'No Emoji Found',
  categories: {
    search: 'Search Results',
    recent: 'Frequently Used',
    people: 'Smileys & People',
    nature: 'Animals & Nature',
    foods: 'Food & Drink',
    activity: 'Activity',
    places: 'Travel & Places',
    objects: 'Objects',
    symbols: 'Symbols',
    flags: 'Flags',
    custom: 'Custom',
  },
}

function makeCustomEmoji(emoji) {
  return Object.assign({}, emoji, {
    id: emoji.short_names[0],
    custom: true
  })
}

export default {
  props: {
    ...PickerProps,
    data: {
      type: Object,
      required: true
    }
  },
  data() {
    let customEmojis = this.custom.map(makeCustomEmoji),
        recentEmojis = this.recent || frequently.get(this.perLine)

    if (recentEmojis.length) {
      recentEmojis = recentEmojis.map((id) => {
        for (let customEmoji of customEmojis) {
          if (customEmoji.id === id) {
            return customEmoji
          }
        }

        return id
      })
    }

    if (this.emojisToShowFilter) {
      customEmojis = customEmojis.filter(e => this.emojisToShowFilter(this.mutableData.emojis[e] || e))
      recentEmojis = recentEmojis.filter(e => this.emojisToShowFilter(this.mutableData.emojis[e] || e))
    }

    return {
      mutableData: this.data.compressed ? uncompress(this.data) : this.data,
      mutableI18n: deepMerge(I18N, this.i18n),
      activeSkin: this.skin || store.get('skin') || this.defaultSkin,
      categories: [],
      activeCategory: null,
      searchEmojis: null,
      customEmojis: customEmojis,
      recentEmojis: recentEmojis,
    }
  },
  computed: {
    customStyles() {
      return {
        width: this.calculateWidth + 'px',
        ...this.pickerStyles
      }
    },
    prefferedTheme() {
      const { theme } = this

      if (theme != 'auto') return theme
      if (typeof window.matchMedia !== 'function') return PickerProps.theme.default

      const darkMatchMedia = matchMedia('(prefers-color-scheme: dark)')

      if (darkMatchMedia.media.match(/^not/)) return PickerProps.theme.default

      return darkMatchMedia.matches ? 'dark' : 'light'
    },
    emojiProps() {
      return {
        native: this.native,
        skin: this.activeSkin,
        size: this.emojiSize,
        set: this.set,
        sheetSize: this.sheetSize,
        forceSize: this.native,
        tooltip: this.emojiTooltip,
        backgroundImageFn: this.backgroundImageFn,
        onClick: this.onEmojiClick.bind(this)
      }
    },
    skinProps() {
      return {
        skin: this.activeSkin
      }
    },
    calculateWidth() {
      return (this.perLine * (this.emojiSize + 12)) + 12 + 2 + measureScrollbar()
    },
    filteredCategories() {
      return this.categories.filter((category) => {
        let isIncluded = this.include && this.include.length ? this.include.indexOf(category.id) > -1 : true
        let isExcluded = this.exclude && this.exclude.length ? this.exclude.indexOf(category.id) > -1 : false
        let hasEmojis = category.emojis.length > 0

        if (this.emojisToShowFilter) {
          hasEmojis = category.emojis.some((emoji) => {
            return this.emojisToShowFilter(this.mutableData.emojis[emoji] || emoji)
          })
        }

        return isIncluded && !isExcluded && hasEmojis
      })
    },
  },
  created() {
    let categories = this.mutableData.categories.map(c => {
      let { id, name, emojis } = c

      if (this.emojisToShowFilter) {
        emojis = c.emojis.filter(e => this.emojisToShowFilter(this.data.emojis[e] || e))
      }

      return { id, name, emojis }
    })

    RECENT_CATEGORY.emojis = this.recentEmojis
    CUSTOM_CATEGORY.emojis = this.customEmojis

    this.categories.push(RECENT_CATEGORY)
    this.categories.push(...categories)
    this.categories.push(CUSTOM_CATEGORY)

    this.categories[0].first = true
    this.activeCategory = this.filteredCategories[0]
  },
  methods: {
    onDarkThemeChange(e) {
      console.log(e)
    },
    onScroll() {
      if (this.infiniteScroll && !this.waitingForPaint) {
        this.waitingForPaint = true
        window.requestAnimationFrame(this.onScrollPaint.bind(this))
      }
    },
    onScrollPaint() {
      this.waitingForPaint = false

      let scrollTop = this.$refs.scroll.scrollTop,
          activeCategory = this.filteredCategories[0]

      for (let i = 0, l = this.filteredCategories.length; i < l; i++) {
        let category = this.filteredCategories[i],
            component = this.$refs.categories[i]

        if (component && component.$el.offsetTop > scrollTop) {
          break
        }

        activeCategory = category
      }

      this.activeCategory = activeCategory
    },
    onAnchorClick(category) {
      let i = this.filteredCategories.indexOf(category),
          component = this.$refs.categories[i],
          scrollToComponent = () => {
            if (component) {
              let top = component.$el.offsetTop

              if (category.first) {
                top = 0
              }

              this.$refs.scroll.scrollTop = top
            }
          }

      if (this.searchEmojis) {
        this.onSearch(null)
        this.$refs.search.clear()

        this.$nextTick(scrollToComponent)
      } else if (this.infiniteScroll) {
        scrollToComponent()
      } else {
        this.activeCategory = this.filteredCategories[i];
      }
    },
    onSearch(emojis) {
      this.searchEmojis = emojis
    },
    onEmojiClick(emoji) {
      this.$emit('select', emoji)
      frequently.add(emoji)
    },
    onSkinChange(skin) {
      this.activeSkin = skin
      store.update({ skin })

      this.$emit('skin-change', skin)
    }
  },
  components: {
    Anchors,
    Category,
    Search
  }
}

</script>
