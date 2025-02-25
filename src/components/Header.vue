<template>
  <header id="header" class="header">
    <div class="header__wrapper">
      <div class="header__title">
        <span class="pair" v-if="pair">{{ pair }}</span>
        <span class="icon-currency"></span>
        <span v-html="price || 'SignificantTrades'"></span>
      </div>
      <dropdown
        :options="timeframes"
        :selected="timeframe"
        @output="setTimeframe(+$event)"
      ></dropdown>
      <!-- <button type="button"
        :class="{active: isReplaying}"
        @click="replay" title="Replay"
        v-tippy="{placement: 'bottom'}">
        <span class="icon-history"></span>
      </button> -->
      <button
        type="button"
        v-if="!isPopupMode"
        @click="togglePopup"
        title="Open as popup"
        v-tippy="{ placement: 'bottom' }"
      >
        <span class="icon-external-link"></span>
      </button>
      <button
        type="button"
        :class="{ active: isSnaped }"
        @click="$store.commit('toggleSnap', !isSnaped)"
        :title="
          isSnaped
            ? 'Disable auto snap'
            : 'Auto snap chart to the latest candle'
        "
        v-tippy="{ placement: 'bottom' }"
      >
        <span class="icon-play"></span>
      </button>
      <button
        type="button"
        :class="{ active: useAudio }"
        @click="$store.commit('toggleAudio', !useAudio)"
      >
        <span class="icon-volume-muted"></span>
      </button>
      <button type="button" @click="$emit('toggleSettings')">
        <span class="icon-cog"></span>
      </button>
    </div>
  </header>
</template>

<script>
import { mapState } from 'vuex'

import socket from '../services/socket'

export default {
  props: ['price'],
  data() {
    return {
      pair: '',
      fetchLabel: 'Fetch timeframe',
      isPopupMode: window.opener !== null,
      canFetch: false,
      showTimeframeDropdown: false,
      timeframeLabel: '15m',
      timeframes: {},
    }
  },
  computed: {
    ...mapState([
      'useAudio',
      'showChart',
      'isSnaped',
      'timeframe',
      'chartRange',
      'isReplaying',
    ]),
  },
  created() {
    this._fetchLabel = this.fetchLabel.substr()
    this.canFetch = socket.canFetch()

    const now = +new Date()

    ;[
      1000 * 10,
      1000 * 30,
      1000 * 60,
      1000 * 60 * 3,
      1000 * 60 * 5,
      1000 * 60 * 10,
    ].forEach((span) => (this.timeframes[span] = this.$root.ago(now - span)))

    socket.$on('pairing', (pair, canFetch) => {
      this.pair = pair

      this.canFetch = canFetch && this.showChart
    })

    socket.$on('fetchStart', () => {
      //
    })

    socket.$on('fetchEnd', () => {
      this.updateTimeframesApproximateContentSize()
    })

    socket.$on('loadingProgress', (event) => {
      if (!event || isNaN(event.progress)) {
        return
      }

      this.fetchLabel = !Math.floor(this.dashoffset)
        ? this._fetchLabel
        : this.sizeOf(event.loaded)
    })

    this.updateTimeframeLabel()
    this.updateTimeframesApproximateContentSize()
  },
  methods: {
    replay() {
      if (this.isReplaying) {
        this.$store.state.isReplaying = false
      } else {
        socket.replay(10)
      }
    },
    setTimeframe(timeframe) {
      document.activeElement.blur()

      this.updateTimeframeLabel(timeframe)

      setTimeout(() => {
        this.$store.commit('setTimeframe', timeframe)
      }, 50)
    },
    updateTimeframeLabel(timeframe) {
      this.timeframeLabel = this.$root.ago(
        +new Date() - (timeframe || this.timeframe)
      )
    },
    togglePopup() {
      window.open(
        window.location.href,
        'Hey hey hey',
        'toolbar=no,status=no,width=350,height=500'
      )

      setTimeout(() => {
        window.close()
      }, 500)
    },
    sizeOf(bytes, si) {
      var thresh = si ? 1000 : 1024
      if (Math.abs(bytes) < thresh) {
        return bytes + ' B'
      }
      var units = si
        ? ['kB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB']
        : ['KiB', 'MiB', 'GiB', 'TiB', 'PiB', 'EiB', 'ZiB', 'YiB']
      var u = -1
      do {
        bytes /= thresh
        ++u
      } while (Math.abs(bytes) > thresh && u < units.length - 1)
      return bytes.toFixed(1) + ' ' + units[u]
    },
    updateTimeframesApproximateContentSize() {
      const now = +new Date()
      const candleCount = this.chartRange / this.timeframe

      if (socket._fetchedTime && socket._fetchedBytes) {
        for (let span in this.timeframes) {
          this.timeframes[span] =
            '<span>~' +
            this.sizeOf(
              socket._fetchedBytes *
                ((span * candleCount) / socket._fetchedTime)
            ) +
            '</span> ' +
            this.$root.ago(now - span).trim()
        }
      }
    },
  },
}
</script>

<style lang="scss">
@import '../assets/sass/variables';

header#header {
  background-color: lighten($dark, 10%);
  color: white;
  position: relative;
  z-index: 2;

  div.header__wrapper {
    position: relative;
    z-index: 1;
    display: flex;
    align-items: center;

    > button,
    .dropdown__selected {
      padding: 0.2em 0.4em;
      font-size: 1em;

      @media screen and (min-width: 480px) {
        padding: 0.2em 0.6em;
      }
    }

    .dropdown__option {
      text-align: left;

      span {
        margin-right: 1em;
      }
    }
  }

  .header__title {
    width: 100%;
    padding: 10px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;

    .pair {
      opacity: 0.5;
    }

    sup {
      line-height: 0;
      opacity: 0.5;

      span {
        font-size: 80%;
        margin-left: 2px;
      }
    }
  }

  .dropdown {
    .options {
      position: absolute;
    }
  }

  button {
    border: 0;
    background: none;
    color: inherit;
    position: relative;
    color: white;

    align-self: stretch;
    cursor: pointer;

    > span {
      display: inline-block;
      transition: all 0.5s $easeElastic;
    }

    .icon-play {
      opacity: 0.2;
    }

    .icon-volume-muted {
      transform: scale(1.2);
    }

    &:hover,
    &:active,
    &.active {
      .icon-external-link {
        transform: rotateZ(-7deg) scale(1.2) translate(5%, -5%);
        text-shadow: 0 0 20px $orange, 0 0 2px white;
      }

      .icon-play {
        opacity: 1;
        transform: rotateZ(-7deg) scale(1.2) translateX(10%);
        text-shadow: 0 0 20px $blue, 0 0 2px white;
      }

      .icon-cog {
        transform: rotateZ(180deg) scale(1.2);
        text-shadow: 0 0 20px $green, 0 0 2px white;
      }

      .icon-history {
        transform: rotateZ(-360deg) scale(1.1);
        display: inline-block;
        text-shadow: 0 0 20px $red, 0 0 2px white;
      }

      .icon-volume-muted {
        color: white;
        transform: rotateZ(-7deg) scale(1.3);
        text-shadow: 0 0 20px $blue, 0 0 2px $blue;
      }
    }

    &.active {
      .icon-play {
        opacity: 1;
        color: $red;
        transform: rotateZ(-7deg) scale(1.2) translateX(10%);
        text-shadow: 0 0 20px $red, 0 0 2px $red;
      }

      .icon-volume-muted {
        &:before {
          content: unicode($icon-volume);
        }
      }

      .icon-history {
        color: white;
        text-shadow: 0 0 20px white, 0 0 2px white;
      }
    }
  }

  &:after,
  &:before {
    content: '';
    position: absolute;
    top: 0;
    left: 0;
    bottom: 0;
    display: block;
    background-clip: padding-box;
    transition: background-color 0.4s $easeOutExpo;
  }
}

#app.loading header#header {
  &:before {
    background-color: lighten($dark, 28%);
    animation: indeterminate-loading-bar-slow 2.1s
      cubic-bezier(0.65, 0.815, 0.735, 0.395) infinite;
  }

  &:after {
    background-color: lighten($dark, 28%);
    animation: indeterminate-loading-bar-fast 2.1s
      cubic-bezier(0.165, 0.84, 0.44, 1) infinite;
    animation-delay: 1.15s;
  }
}

@keyframes indeterminate-loading-bar-slow {
  0% {
    left: -35%;
    right: 100%;
  }

  60% {
    left: 100%;
    right: -90%;
  }

  100% {
    left: 100%;
    right: -90%;
  }
}

@keyframes indeterminate-loading-bar-fast {
  0% {
    left: -200%;
    right: 100%;
  }

  60% {
    left: 107%;
    right: -8%;
  }

  100% {
    left: 107%;
    right: -8%;
  }
}
</style>
