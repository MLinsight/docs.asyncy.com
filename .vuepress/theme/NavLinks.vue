<template>
  <nav class="nav-links" v-if="userLinks.length || repoLink">
    <!-- user links -->
    <div
      class="nav-item"
      v-for="item in userLinks"
      :key="item.link">
      <DropdownLink v-if="item.type === 'links'" :item="item"/>
      <NavLink v-else :item="item"/>
    </div>
    <!-- repo link -->
    <div class="nav-item nav-repo">
      <a v-if="repoLink"
        :href="repoLink"
        class="repo-link"
        target="_blank"
        rel="noopener">
        <github-icon class="github" />
        <github-icon black class="github s" />
      </a>
    </div>
  </nav>
</template>

<script>
import OutboundLink from './OutboundLink.vue'
import DropdownLink from './DropdownLink.vue'
import { resolveNavLinkItem } from './util'
import NavLink from './NavLink.vue'
import GithubIcon from './github.vue'

export default {
  components: { OutboundLink, NavLink, DropdownLink, GithubIcon },
  computed: {
    userNav () {
      return this.$themeLocaleConfig.nav || this.$site.themeConfig.nav || []
    },
    nav () {
      const { locales } = this.$site
      if (locales && Object.keys(locales).length > 1) {
        const currentLink = this.$page.path
        const routes = this.$router.options.routes
        const themeLocales = this.$site.themeConfig.locales || {}
        const languageDropdown = {
          text: this.$themeLocaleConfig.selectText || 'Languages',
          items: Object.keys(locales).map(path => {
            const locale = locales[path]
            const text = themeLocales[path] && themeLocales[path].label || locale.lang
            let link
            // Stay on the current page
            if (locale.lang === this.$lang) {
              link = currentLink
            } else {
              // Try to stay on the same page
              link = currentLink.replace(this.$localeConfig.path, path)
              // fallback to homepage
              if (!routes.some(route => route.path === link)) {
                link = path
              }
            }
            return { text, link }
          })
        }
        return [...this.userNav, languageDropdown]
      }
      return this.userNav
    },
    userLinks () {
      return (this.nav || []).map(link => {
        return Object.assign(resolveNavLinkItem(link), {
          items: (link.items || []).map(resolveNavLinkItem)
        })
      })
    },
    repoLink () {
      const { repo } = this.$site.themeConfig
      if (repo) {
        return /^https?:/.test(repo)
          ? repo
          : `https://github.com/${repo}`
      }
    },
    repoLabel () {
      if (!this.repoLink) return
      if (this.$site.themeConfig.repoLabel) {
        return this.$site.themeConfig.repoLabel
      }

      const repoHost = this.repoLink.match(/^https?:\/\/[^/]+/)[0]
      const platforms = ['GitHub', 'GitLab', 'Bitbucket']
      for (let i = 0; i < platforms.length; i++) {
        const platform = platforms[i]
        if (new RegExp(platform, 'i').test(repoHost)) {
          return platform
        }
      }

      return 'Source'
    }
  }
}
</script>

<style lang="stylus">
@import './styles/config.styl'

.nav-links
  display inline-flex
  height 100%
  a
    font-weight normal
    font-size 1.05em
    line-height 1.4rem
    color inherit
    &:hover, &.router-link-active
      color $accentColor
  .nav-item
    cursor pointer
    position relative
    display inline-flex
    align-items center
    margin-left 4rem
    height 100%
  .repo-link
    position relative
    margin 0
    padding 0
    color transparent
    width 30px
    .github
      // background-image url(./github.svg?color=black)
      &.s
        display none
      background-repeat no-repeat
      background-size contain
      display inline-block
      min-width 24px
      min-height 24px
    &:hover
      color transparent

@media (max-width: $MQMobile)
  .nav-links
    height auto
    .nav-item, .repo-link
      margin-left 0
      height auto
    .nav-repo
      display none !important

@media (min-width: $MQMobile)
  .nav-links a
    &:hover, &.router-link-active
      color $headerLinkHoverColor
</style>
