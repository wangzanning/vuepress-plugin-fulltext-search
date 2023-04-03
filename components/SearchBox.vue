<template>
  <div class="search-box">
    <input
      ref="input"
      aria-label="Search"
      :value="query"
      :class="[{ focused: focused }, 'searchInput']"
      :placeholder="placeholder"
      autocomplete="off"
      spellcheck="false"
      @input="handleInput($event)"
      @focus="focused = true"
      @blur="focused = false"
      @keyup.enter="go(focusIndex)"
      @keyup.up="onUp"
      @keyup.down="onDown"
    />
    <ul v-if="showSuggestions" class="suggestions" :class="{ 'align-right': alignRight }" @mouseleave="unfocus"
        ref="suggestions">
      <li class="total-suggestions">共搜索到{{ suggestions.length || 0 }}条内容</li>
      <li
        v-for="(s, i) in suggestions"
        :key="i"
        :ref="`item_${i}`"
        class="suggestion"
        :class="{ focused: i === focusIndex }"
        @mousedown="go(i)"
        @mouseenter="focus(i)"
      >
        <a :href="s.path + s.slug" @click.prevent>
          <div v-if="s.relativePath" class="parent-page-title" v-html="getParentTitle(s.relativePath)"/>
          <div class="suggestion-row">
            <div class="suggestion-content">
              <!-- prettier-ignore -->
              <div v-if="s.contentStr">
                {{ s.contentDisplay.prefix }}<span class="highlight">{{
                  s.contentDisplay.highlightedContent
                }}</span>{{ s.contentDisplay.suffix }}
              </div>
              <div v-else>
                {{ s.headingDisplay.prefix }}<span class="highlight">{{
                  s.headingDisplay.highlightedContent
                }}</span>{{ s.headingDisplay.suffix }}
              </div>
            </div>
          </div>
        </a>
      </li>
    </ul>
  </div>
</template>

<script>
import flexsearchSvc from '../services/flexsearchSvc';
import _ from 'lodash';

// see https://vuepress.vuejs.org/plugin/option-api.html#clientdynamicmodules
import hooks from '@dynamic/hooks'

/* global SEARCH_MAX_SUGGESTIONS, SEARCH_PATHS, SEARCH_HOTKEYS */
export default {
  name: 'SearchBox',
  data() {
    return {
      query: '',
      focused: false,
      focusIndex: 0,
      placeholder: undefined,
      suggestions: null,
      highlightWord: '',
      lastWord: '',
    }
  },
  computed: {
    queryTerms() {
      if (!this.query) return []
      const result = flexsearchSvc
        .normalizeString(this.query)
        .split(/[^\p{L}\p{N}_]+/iu)
        .filter(t => t)
      return result
    },
    showSuggestions() {
      return this.focused && this.suggestions && this.suggestions.length
    },

    // make suggestions align right when there are not enough items
    alignRight() {
      const navCount = (this.$site.themeConfig.nav || []).length
      const repo = this.$site.repo ? 1 : 0
      return navCount + repo <= 2
    },
  },
  watch: {
    query(val) {
      this.getSuggestions()
      if (!this.suggestions?.length && this.lastWord) {
        this.$nextTick(() => {
          let target = document.querySelector('.theme-default-content.content__default').innerHTML
          document.querySelector('.theme-default-content.content__default').innerHTML = target.replace(`<font>${this.lastWord}</font>`, this.lastWord)
        })
      }
    },
    $route(to, from) {
      let val = this.query;
      this.$nextTick(() => {
        let target = document.querySelector('.theme-default-content.content__default').innerHTML
        if (this.query && target) {
          // reg: val not contained in html tag
          const reg = new RegExp(`(?<!<[^>]*?)${val}(?![^<]*?>)`, 'gi')
          document.querySelector('.theme-default-content.content__default').innerHTML = target.replace(reg, `<font>${val}</font>`)
          this.lastWord = val // 记录上一次查询词语
        }
      });
    }
  },
  /* global OPTIONS */
  mounted() {
    const options = OPTIONS || {}
    flexsearchSvc.buildIndex(this.$site.pages, options)
    this.placeholder = this.$site.themeConfig.searchPlaceholder || ''
    document.addEventListener('keydown', this.onHotkey)

    // set query from URL
    const params = this.urlParams()
    if (params) {
      const query = params.get('query')
      if (query) {
        this.query = decodeURI(query)
        this.focused = true
      }
    }
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this.onHotkey)
  },
  methods: {
    async getSuggestions() {
      if (!this.query || !this.queryTerms.length) {
        this.suggestions = []
        return
      }
      let suggestions = await flexsearchSvc.match(
        this.query,
        this.queryTerms,
        this.$site.themeConfig.searchMaxSuggestions || SEARCH_MAX_SUGGESTIONS,
      )
      if (hooks.processSuggestions) {
        // augment suggestions with user-provided function
        suggestions = await hooks.processSuggestions(suggestions, this.query, this.queryTerms)
      }
      this.suggestions = suggestions.map(s => ({
        ...s,
        headingDisplay: highlight(s.headingStr, s.headingHighlight),
        contentDisplay: highlight(s.contentStr, s.contentHighlight),
      }))
      // 按每层目录顺序排序，由最内层至最外层
      this.suggestions = _.sortBy(this.suggestions, this.sortEachToc(4));
      this.suggestions = _.sortBy(this.suggestions, this.sortEachToc(3));
      this.suggestions = _.sortBy(this.suggestions, this.sortEachToc(2));
      this.suggestions = _.sortBy(this.suggestions, this.sortEachToc(1));
      this.suggestions = _.sortBy(this.suggestions, this.sortEachToc(0));
    },
    sortEachToc(layer) {
      return function (n){
        let temp = n.relativePath.split('/')[layer]?.replace(/[^\d+]/g, '') || '999';
        while (temp.length < 3) {
          temp = temp + '0' ;
        }
        return temp;
      }
    },
    getPageLocalePath(page) {
      for (const localePath in this.$site.locales || {}) {
        if (localePath !== '/' && page.path.indexOf(localePath) === 0) {
          return localePath
        }
      }
      return '/'
    },
    isSearchable(page) {
      let searchPaths = SEARCH_PATHS
      // all paths searchables
      if (searchPaths === null) {
        return true
      }
      searchPaths = Array.isArray(searchPaths) ? searchPaths : new Array(searchPaths)
      return (
        searchPaths.filter(path => {
          return page.path.match(path)
        }).length > 0
      )
    },
    onHotkey(event) {
      if (event.srcElement === document.body && SEARCH_HOTKEYS.includes(event.key)) {
        this.$refs.input.focus()
        event.preventDefault()
      }
    },
    onUp() {
      if (this.showSuggestions) {
        if (this.focusIndex > 0) {
          this.focusIndex--
        } else {
          this.focusIndex = this.suggestions.length - 1
        }
      }
      // this.ensureFocusedInView(this.focusIndex);
      this.showCurrentResult(this.focusIndex);
    },
    onDown() {
      if (this.showSuggestions) {
        if (this.focusIndex < this.suggestions.length - 1) {
          this.focusIndex++
        } else {
          this.focusIndex = 0
        }
      }
      this.focused = true
      // this.ensureFocusedInView(this.focusIndex);
      this.showCurrentResult(this.focusIndex);
    },
    go(i) {
      this.highlightWord = ''
      if (!this.showSuggestions) {
        return
      }
      // debugger
      if (hooks.onGoToSuggestion) {
        const result = hooks.onGoToSuggestion(i, this.suggestions[i], this.query, this.queryTerms)
        if (result === true) return
      }
      if (this.suggestions[i]?.external) {
        window.open(this.suggestions[i].path + this.suggestions[i].slug, '_blank')
      } else {
        this.$router.push(this.suggestions[i].path + this.suggestions[i].slug)
        // this.query = ''
        const val = this.suggestions[i]?.contentDisplay?.highlightedContent
        this.highlightWord = val || ''
        // reset query param
        const params = this.urlParams()
        if (params) {
          params.delete('query')
          const paramsString = params.toString()
          const newState = window.location.pathname + (paramsString ? `?${paramsString}` : '')
          history.pushState(null, '', newState)
        }
      }
      this.focusIndex = 0
      this.focused = false
    },
    focus(i) {
      this.focusIndex = i
      this.showCurrentResult(i);
    },
    unfocus() {
      this.focusIndex = -1
    },
    showCurrentResult(i) {
      if (!this.showSuggestions) {
        return
      }
      if (hooks.onGoToSuggestion) {
        const result = hooks.onGoToSuggestion(i, this.suggestions[i], this.query, this.queryTerms)
        if (result === true) return
      }
      if (this.suggestions[i].external) {
        window.open(this.suggestions[i].path + this.suggestions[i].slug, '_blank')
      } else {
        this.$router.push(this.suggestions[i].path + this.suggestions[i].slug)
        const val = this.suggestions[i]?.contentDisplay?.highlightedContent
        this.highlightWord = val || ''
        this.focused = true;

        // reset query param
        const params = this.urlParams()
        if (params) {
          params.delete('query')
          const paramsString = params.toString()
          const newState = window.location.pathname + (paramsString ? `?${paramsString}` : '')
          history.pushState(null, '', newState)
        }
      }
    },
    urlParams() {
      if (!window.location.search) {
        return null
      }
      return new URLSearchParams(window.location.search)
    },
    ensureFocusedInView(index, natural) {
      if (!this.showSuggestions) {
        return
      }
      const focusEl = this.$refs[`item_${this.focusIndex}`][0];
      const parentEl = this.$refs.suggestions;
      console.log('el', focusEl)
      console.log('focusElRECT', focusEl.getBoundingClientRect());
      console.log('parentEl', parentEl.getBoundingClientRect());

    },
    handleInput(event) {
      this.query = event.target.value
      this.focused = true
    },
    getParentTitle(path) {
      const parentPath = path.split('/').map((item)=>{
        return item.replace(/(^\d+\.)/, '').replace(/(\.md$)/, '');
      }).join(' > ');
      return parentPath;
    }
  },
}

function highlight(str, strHighlight) {
  if (!str) return {}
  if (!strHighlight) return {prefix: str}
  const [start, length] = strHighlight
  const end = start + length

  const prefix = str.slice(0, start)
  const highlightedContent = str.slice(start, end)
  const suffix = str.slice(end)
  return {prefix, highlightedContent, suffix}

  // return `${prefix}<span class="highlight">${highlightedContent}</span>${suffix}`
}

</script>

<style lang="stylus">
font
  background yellow

  searchInput
    border-radius 4px

.search-box
  display inline-block
  position relative
  margin-right 1rem

  .total-suggestions
    margin 1px 0 2x 2px
    font-family 'PingFang SC';
    font-style normal;
    font-weight 400;
    font-size 14px;
    color: #666666;

  input
    cursor text
    width 13rem
    height: 2rem
    color lighten($textColor, 25%)
    display inline-block
    border 1px solid darken($borderColor, 10%)
    border-radius 4px
    font-size 0.9rem
    line-height 2rem
    padding 0 0.5rem 0 0.5rem
    outline none
    transition all .2s ease
    background #fff url(../assets/search.svg) no-repeat
    background-position right 5% center
    background-size 1rem

    &:focus
      cursor auto
      border-color $accentColor

  .suggestions
    background #fff
    min-width 500px
    max-width 700px
    position absolute
    top 2rem
    right 0rem
    border 1px solid darken($borderColor, 10%)
    border-radius 6px
    padding 0.4rem
    list-style-type none

    &.align-right
      right 0

  .suggestion
    line-height 1.4
    // padding 0.4rem 0.6rem
    border-radius 4px
    cursor pointer
    width 100%

    &.focused
      background-color #eee

    a
      display block
      white-space normal
      color lighten($textColor, 15%)
      width 100%

      .parent-page-title
        color #222222
        font-family 'PingFang SC';
        font-style: normal;
        font-weight 500;
        font-size: 14px;
        background-color $accentColor
        padding 5px

      .suggestion-row
        border-collapse collapse
        width 100%
        display table

        .page-title
          width: 35%
          border 1px solid $borderColor
          border-left none
          display table-cell
          text-align right
          padding 5px
          font-weight 600

        .suggestion-content
          .highlight
            text-decoration: underline
          border 1px solid $borderColor
          font-weight 400
          border-right none
          width: 65%
          display table-cell
          padding 5px

          .header
            font-weight 600

@media (max-width: $MQNarrow)
  .search-box
    input
      cursor pointer
      width 0
      border-color transparent
      position relative

      &:focus
        cursor text
        left 0
        width 10rem

// Match IE11
@media all and (-ms-high-contrast: none)
  .search-box input
    height 2rem

@media (max-width: $MQNarrow) and (min-width: $MQMobile)
  .search-box
    .suggestions
      left 0

@media (max-width: $MQMobile)
  .search-box
    margin-right 0

    input
      left 1rem

    .suggestions
      right 0

@media (max-width: $MQMobileNarrow)
  .search-box
    .suggestions
      width calc(100vw - 4rem)

    input:focus
      width 8rem
</style>
