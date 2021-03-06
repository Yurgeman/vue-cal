<template lang="pug">
  .vuecal__cell(:class="[cssClass, splits.length ? 'splitted' : '']" :style="cellStyles")
    .vuecal__cell-content(:class="splits.length && `vuecal__cell-split ${splits[i - 1].class}`" v-for="i in (splits.length || 1)")
      .split-label(v-if="splits.length" v-html="splits[i - 1].label")
      div(v-if="content" v-html="content")
      div(v-else)
        .vuecal__no-event(v-if="!events.length") {{ texts.noEvent }}
        .vuecal__event(:class="eventClasses(event)"
                       v-else
                       v-for="(event, j) in (splits.length ? splitEvents[i] : events)" :key="j"
                       :style="eventStyles(event)"
                       @click.stop="focusEvent(event)"
                       @mousedown.stop="onMouseDown($event, event)")
          .vuecal__event-delete(v-if="$parent.editableEvents" @click.stop.prevent="deleteEvent(event)") {{ texts.deleteEvent }}
          .vuecal__event-title.vuecal__event-title--edit(contenteditable v-if="$parent.editableEvents && event.title" @blur="event.title = $event.target.innerHTML" v-html="event.title")
          .vuecal__event-title(v-else-if="event.title") {{ event.title }}
          .vuecal__event-time(v-if="event.startTime")
            | {{ event.startTime }}
            span(v-if="event.endTime") &nbsp;- {{ event.endTime }}
          .vuecal__event-content(v-if="event.content" v-html="event.content")
          .vuecal__event-resize-handle(v-if="$parent.editableEvents && event.startTime"
                                       @mousedown.stop="$parent.editableEvents && $parent.time && onDragHandleMouseDown($event, event)"
                                       @touchstart.stop="$parent.editableEvents && $parent.time && onDragHandleMouseDown($event, event)")
</template>

<script>
export default {
  props: {
    cssClass: {
      type: String,
      default: ''
    },
    date: {
      type: Date,
      required: true
    },
    formattedDate: {
      type: String,
      default: ''
    },
    content: {
      type: [String, Number],
      default: ''
    },
    splits: {
      type: Array,
      default: () => []
    }
  },
  data: () => ({
    splitEvents: [],
    comparisonArray: {}
  }),

  methods: {
    updateEventPosition (event) {
      let minutesFromTop = event.startTimeMinutes - this.$parent.timeFrom
      const top = Math.round(minutesFromTop * this.timeCellHeight / this.timeStep)

      minutesFromTop = event.endTimeMinutes - this.$parent.timeFrom
      const bottom = Math.round(minutesFromTop * this.timeCellHeight / this.timeStep)

      event.top = top
      event.height = bottom - top
    },

    eventStyles (event) {
      if (!event.startTime) return {}
      const resizeAnEvent = this.domEvents.resizeAnEvent

      return {
        top: `${event.top}px`,
        height: `${resizeAnEvent.newHeight && resizeAnEvent.eventId === event.id ? resizeAnEvent.newHeight : event.height}px`
      }
    },

    eventClasses (event) {
      return {
        ...event.classes,
        'vuecal__event--focus': this.domEvents.focusAnEvent.eventId === event.id,
        'vuecal__event--deletable': this.domEvents.clickHoldAnEvent.eventId === event.id
      }
    },

    checkCellOverlappingEvents () {
      this.events.forEach(event => {
        if (!event.background) {
          // Unique comparison of events.
          this.comparisonArray[event.id] = this.events.filter(item => (item.id !== event.id && Object.keys(this.comparisonArray).indexOf(item.id) === -1 && !item.background))
          if (this.comparisonArray[event.id].length) this.checkOverlappingEvents(event)
        }
      })
    },

    checkOverlappingEvents (event) {
      this.comparisonArray[event.id].forEach((event2, i) => {
        const event1startsFirst = event.startTimeMinutes < event2.startTimeMinutes
        const event1overlapsEvent2 = event1startsFirst && event.endTimeMinutes > event2.startTimeMinutes
        const event2overlapsEvent1 = !event1startsFirst && event2.endTimeMinutes > event.startTimeMinutes
        // console.log('comparing event ' + event.title + ' with ', event2.title)
        if (event1overlapsEvent2 || event2overlapsEvent1) {
          event.classes = Object.assign(event.classes, {
            'vuecal__event--overlapped': event1startsFirst,
            'vuecal__event--overlapping': !event1startsFirst
          })
          event2.classes = Object.assign(event2.classes, {
            'vuecal__event--overlapped': !event1startsFirst,
            'vuecal__event--overlapping': event1startsFirst
          })
          // If up to 3 events start at the same time.
          // if (event.classes.startTimeMinutes === event2.classes.startTimeMinutes) {
          //   event.classes.split3 = event.classes.split2
          //   event.classes.split2 = true
          //   event2.classes.split3 = event2.classes.split2
          //   event2.classes.split2 = true
          //   event2.classes.splitm = event.classes.split2 && !event2.classes.middle
          // }
        } else {
          // event.classes.overlapped = Math.max(event.classes.overlapped - 1, 0)
          // event.classes.overlapping = Math.max(event.classes.overlapping - 1, 0)
          // event2.classes.overlapped = Math.max(event2.classes.overlapped - 1, 0)
          // event2.classes.overlapping = Math.max(event2.classes.overlapping - 1, 0)
        }
        // console.log({event1startsFirst, event1overlapsEvent2, event2overlapsEvent1, overlapping: event.classes['vuecal__event--overlapping'], overlapped: event.classes['vuecal__event--overlapped']})
      })
    },

    onResizeEvent (eventId, newHeight) {
      let event = this.events.filter(e => e.id === eventId)[0]
      if (event) {
        event.height = Math.max(newHeight, 10)
        this.updateEndTimeOnResize(event)

        // if (!event.background) {
        //   this.checkOverlappingEvents(event)
        // }
      }
    },

    updateEndTimeOnResize (event) {
      const bottom = event.top + event.height
      const endTime = (bottom / this.timeCellHeight * this.timeStep + this.timeFrom) / 60
      const hours = parseInt(endTime)
      const minutes = parseInt((endTime - hours) * 60)

      event.endTimeMinutes = endTime * 60
      event.endTime = `${hours}:${(minutes < 10 ? '0' : '') + minutes}`
    },

    // On an event.
    onMouseDown (e, event) {
      let clickHold = this.domEvents.clickHoldAnEvent
      clickHold.timeoutId = setTimeout(() => {
        // Disable delete feature on mobile as not yet ready.
        if (!('ontouchstart' in window)) clickHold.eventId = event.id
        window.removeEventListener('ontouchstart' in window ? 'touchend' : 'mouseup', this.onCancelClickHold, { once: true })
      }, clickHold.timeout)
      window.addEventListener('ontouchstart' in window ? 'touchend' : 'mouseup', this.onCancelClickHold, { once: true })
    },

    onCancelClickHold () {
      let clickHold = this.domEvents.clickHoldAnEvent
      clickHold.eventId = null
      clearTimeout(clickHold.timeoutId)
    },

    onDragHandleMouseDown (e, event) {
      const start = 'ontouchstart' in window && e.touches ? e.touches[0].clientY : e.clientY
      this.domEvents.resizeAnEvent = Object.assign(this.domEvents.resizeAnEvent, { start, originalHeight: event.height, newHeight: event.height, eventId: event.id })
    },

    deleteEvent (event) {
      this.events = this.events.filter(e => e.id !== event.id)

      if (this.splits.length) this.splitEvents = this.splitEvents[event.split].filter(e => e.id !== event.id)

      // this.checkCellOverlappingEvents()
    },

    focusEvent (event) {
      this.domEvents.focusAnEvent.eventId = event.id
    }
  },

  computed: {
    texts () {
      return this.$parent.texts
    },
    timeCellHeight () {
      return parseInt(this.$parent.timeCellHeight)
    },
    timeFrom () {
      return parseInt(this.$parent.timeFrom)
    },
    timeStep () {
      return parseInt(this.$parent.timeStep)
    },
    domEvents: {
      get () {
        if (this.$parent.domEvents.resizeAnEvent.eventId) {
          this.onResizeEvent(this.$parent.domEvents.resizeAnEvent.eventId, this.$parent.domEvents.resizeAnEvent.newHeight)
        }
        return this.$parent.domEvents
      },
      set (object) {
        this.$parent.domEvents = object
      }
    },
    cellStyles () {
      return { minWidth: `${this.$parent.minCellWidth}px` || null }
    },
    events: {
      get () {
        const events = this.$parent.mutableEvents[this.formattedDate]
        // eslint-disable-next-line
        this.splitEvents = []

        return events ? events.map(event => {
          if (event.startTime) this.updateEventPosition(event)

          // Only for splits.
          if (this.splits.length && event.split) {
            // eslint-disable-next-line
            if (!this.splitEvents[event.split]) this.$set(this.splitEvents, event.split, [])
            // eslint-disable-next-line
            this.splitEvents[event.split].push(event)
          }

          if (!event.background) {
            // Unique comparison of events.
            // eslint-disable-next-line
            this.comparisonArray[event.id] = events.filter(item => (item.id !== event.id && Object.keys(this.comparisonArray).indexOf(item.id) === -1 && !item.background))
            if (this.comparisonArray[event.id].length) this.checkOverlappingEvents(event)
          }

          return event
        }) : []
      },
      set (events) {
        this.$parent.mutableEvents[this.formattedDate] = events
      }
    }
  }
}
</script>

<style lang="scss">
.vuecal__cell {
  width: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
  position: relative;
  .vuecal--month-view &,
  .vuecal--week-view &,
  .vuecal--day-view & {
    width: 14.2857%;
  }

  .vuecal--hide-weekends.vuecal--month-view &,
  .vuecal--hide-weekends.vuecal--week-view &,
  .vuecal--hide-weekends.vuecal--day-view & {
    width: 20%;
  }

  .vuecal--years-view & {width: 20%;}
  .vuecal--year-view & {width: 33.33%;}
  .vuecal--day-view & {flex: 1;}

  .vuecal--click-to-navigate & {cursor: pointer;}
  .vuecal--view-with-time &,
  .vuecal--week-view.vuecal--no-time &,
  .vuecal--day-view.vuecal--no-time & {display: block;}

  &.splitted {
    flex-direction: row;
    display: flex;
  }

  .vuecal__cell-split {
    display: flex;
    flex-grow: 1;
    flex-direction: column;
    height: 100%;
    position: relative;
  }

  &:before {
    position: absolute;
    top: 0;
    left: 0;
    right: -1px;
    bottom: -1px;
    border: 1px solid #ddd;
    content: '';

    .vuecal--day-view & {left: 1px;right: 0;bottom: 0;}
    .vuecal--view-with-time.vuecal--day-view & {left: 0;}
  }

  &.today,
  &.current {
    background-color: rgba(240, 240, 255, 0.4);
    z-index: 1;
  }

  &.selected {
    background-color: rgba(235, 255, 245, 0.4);
    z-index: 2;

    // .vuecal--day-view &:before {background: none;border: 1px solid rgba(235, 255, 245, 0.4);}
    .vuecal--day-view & {background: none;}
  }

  &.out-of-scope {
    color: #ccc;
  }
}

.vuecal--split-days.vuecal--week-view .vuecal__cell.splitted {
  overflow: hidden;
}

.vuecal__no-event {
  padding-top: 1em;
  color: #aaa;
  user-select: none;
}

// EVENTS.
//======================================//
.vuecal__event {
  color: #666;
  background-color: #f8f8f8;
  position: relative;
  z-index: 1;
  transition: box-shadow 0.3s;
  overflow: hidden;// For sliding delete button.

  &:hover {z-index: 2;}

  // Reactivate user selection in events.
  .vuecal__cell & * {user-select: auto;}

  .vuecal--view-with-time & {
    position: absolute;
    left: 0;
    right: 0;

    // &:hover {height: auto !important;}
  }

  &.vuecal__event--overlapped {right: 20%;}
  &.vuecal__event--overlapping:not(.split2):not(.split3) {left: 30%;box-shadow: 0 0 5px rgba(#000, 0.2);}
  &.vuecal__event--overlapped.split2 {right: 50%;}
  &.vuecal__event--overlapping.split2 {left: 50%;}
  &.vuecal__event--overlapped.split3 {right: 66.66%;}
  &.vuecal__event--overlapping.split3 {left: 66.66%;}
  &.vuecal__event--overlapping.split3.split-middle {left: 33.33%;right: 33.33%;}
  &.vuecal__event--background {z-index: 0;}
  &.vuecal__event--focus {box-shadow: 1px 1px 6px rgba(0,0,0,0.2);z-index: 3;}
}

.vuecal__event-resize-handle {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 1em;
  background-color: rgba(255, 255, 255, 0.3);
  opacity: 0;
  transform: translateY(110%);
  transition: 0.3s;
  cursor: ns-resize;

  .vuecal__event:hover & {opacity: 1;transform: translateY(0);}
}

.vuecal__event-delete {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: 1.4em;
  line-height: 1.4em;
  background-color: rgba(221, 51, 51, 0.85);
  color: #fff;
  z-index: 1;
  cursor: pointer;
  transform: translateY(-110%);
  transition: 0.3s;

  .vuecal--full-height-delete & {
    height: auto;
    bottom: 0;

    &:before {
      content: '';
      width: 1.7em;
      height: 1.8em;
      display: block;
      background-image: url('data:image/svg+xml;utf8,<svg width="512" height="512" viewBox="0 0 512 512" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"><path d="m256 33c-124 0-224 100-224 224 0 124 100 224 224 224 124 0 224-100 224-224 0-124-100-224-224-224z m108 300c2 1 3 3 3 5 0 2-1 4-3 6l-21 21c-2 2-4 3-6 3-2 0-4-1-5-3l-76-75-75 76c-2 1-4 2-6 2-2 0-4-1-6-2l-21-22c-2-2-2-4-2-6 0-2 0-4 2-5l76-76-76-75c-3-3-3-9 0-12l21-21c2-2 4-3 6-3 2 0 4 1 5 3l76 74 76-74c1-2 3-3 5-3 3 0 5 1 6 3l22 21c3 3 3 9 0 12l-76 75z" transform="scale(0.046875 0.046875)" fill="#fff" opacity="0.9"/></svg>');
    }
  }
  .vuecal__event--deletable & {transform: translateY(0);}
}

.vuecal__event-title--edit {
  border-bottom: 1px solid transparent;
  text-align: center;
  transition: 0.3s;
  color: inherit;
  background-image: url('data:image/svg+xml;utf8,<svg width="512" height="512" viewBox="0 0 512 512" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"><path d="m163 440l-91-91 251-250 90 90z m309-352l-48-48c-12-11-32-11-45 2l-45 45 91 91 45-45c13-13 13-33 2-45z m-408 275l-32 117 117-32z" fill="#000" opacity="0.4"/></svg>');
  background-repeat: no-repeat;
  background-position: 120% 0.15em;
  background-size: 0.4em;
  outline: none;
  width: 100%;

  &:hover, &:focus {
    border-color: rgba(0, 0, 0, 0.4);
    background-position: 99% 0.15em;
    background-size: 1.2em;
  }
}
</style>
