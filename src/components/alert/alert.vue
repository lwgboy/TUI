<template>
  <transition name="t-fade-out">
    <div class="t-alert" :class="[
    type ? `t-alert--${type}` : ''
  ]" v-if="!hide">
      <div class="t-alert__title" :class="[
        textCenter ? 't-alert__title--center' : ''
      ]">
        <i v-if="showIcon" class="t-alert__title-icon" :class="icon || iconClasses[type]"></i>
        {{ content }}
      </div>
      <div class="t-alert__sub" v-if="sub">
        {{ sub }}
      </div>
      <template v-if="closable">
        <span class="t-alert__close t-alert__close-text" v-if="closeText" @click="$emit('alert-close')">{{ closeText }}</span>
        <i class="t-alert__close t-alert__close--icon fa fa-times" v-if="!closeText" @click="$emit('alert-close')"></i>
      </template>
    </div>
  </transition>
</template>

<script>
export default {
  name: 't-alert',

  data () {
    return {
      hide: false,
      iconClasses: {
        'success': 'fa fa-check-circle',
        'info': 'fa fa-info-circle',
        'danger': 'fa fa-times-circle',
        'warning': 'fa fa-exclamation-circle'
      }
    }
  },

  props: {
    content: String,
    sub: String,
    type: String,
    closable: {
      type: Boolean,
      default: true
    },
    closeText: String,
    showIcon: Boolean,
    textCenter: Boolean,
    icon: String
  },

  created () {
    this.$on('alert-close', this.closeHandler)
  },

  methods: {
    closeHandler () {
      this.hide = true
      this.$emit('close')
    }
  }
}
</script>
