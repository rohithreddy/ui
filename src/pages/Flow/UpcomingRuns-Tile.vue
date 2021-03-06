<script>
import moment from 'moment-timezone'
import { mapGetters } from 'vuex'

import Alert from '@/components/Alert'
import CardTitle from '@/components/Card-Title'
import ConcurrencyInfo from '@/components/ConcurrencyInfo'
import DurationSpan from '@/components/DurationSpan'
import { cancelLateRunsMixin } from '@/mixins/cancelLateRunsMixin'
import { runFlowNowMixin } from '@/mixins/runFlowNow'

export default {
  components: {
    Alert,
    CardTitle,
    ConcurrencyInfo,
    DurationSpan
  },
  mixins: [cancelLateRunsMixin, runFlowNowMixin],
  props: {
    aggregate: {
      type: Boolean,
      default: () => false
    },
    flow: {
      type: Object,
      default: () => {}
    },
    fullHeight: {
      required: false,
      type: Boolean,
      default: () => false
    }
  },
  data() {
    return { loading: 0, tab: 'upcoming', setToRun: [] }
  },
  computed: {
    ...mapGetters('api', ['isCloud']),
    ...mapGetters('user', ['timezone']),
    lateRuns() {
      if (!this.upcoming) return []
      return this.upcoming.filter(run => {
        return (
          this.getTimeOverdue(run.scheduled_start_time)._milliseconds > 20000
        )
      })
    },
    pollInterval() {
      return this.flow.archived ? 0 : 5000
    },
    upcomingRuns() {
      if (!this.upcoming) return []
      return this.upcoming.filter(run => {
        return (
          this.getTimeOverdue(run.scheduled_start_time)._milliseconds <= 20000
        )
      })
    },
    title() {
      if (this.flow.archived) {
        return 'Upcoming Runs'
      }

      let title = ''

      if (this.tab == 'upcoming') {
        title =
          this.loading > 0
            ? 'Upcoming Runs'
            : `${this.upcomingRuns?.length} Upcoming Runs`
      }

      if (this.tab == 'late') {
        title =
          this.loading > 0 || this.isClearingLateRuns
            ? 'Late Runs'
            : `${this.lateRuns?.length} Late Runs`
      }

      return title
    },
    titleIcon() {
      let icon = ''

      if (this.tab == 'upcoming') {
        icon = 'access_time'
      }

      if (this.tab == 'late') {
        icon = 'timelapse'
      }

      return icon
    },
    titleIconColor() {
      return this.flow.archived || this.loading > 0
        ? 'grey'
        : this.tab == 'upcoming'
        ? 'primary'
        : this.lateRuns.length > 0
        ? 'deepRed'
        : 'Success'
    }
  },
  watch: {
    upcoming(val) {
      if (!val) return
      if (this.lateRuns.length > 0) {
        this.tab = 'late'
      }
    },
    flow() {
      this.$apollo.queries.upcoming.stopPolling()

      if (this.pollInterval > 0) {
        this.$apollo.queries.upcoming.startPolling(this.pollInterval)
      } else {
        this.$apollo.queries.upcoming.refetch()
      }
    }
  },
  mounted() {
    if (this.pollInterval > 0) {
      this.$apollo.queries.upcoming.startPolling(this.pollInterval)
    }
  },
  methods: {
    formatDate(timestamp) {
      if (!timestamp) throw new Error('Did not receive a timestamp')

      let t = moment(timestamp).tz(this.timezone),
        shortenedTz = moment()
          .tz(this.timezone || Intl.DateTimeFormat().resolvedOptions().timeZone)
          .zoneAbbr()

      let timeObj = t ? t : moment(timestamp)

      let formatted = timeObj.format('MMMM D, YYYY [at] h:mma')
      return `${formatted} ${shortenedTz}`
    },
    formatTime(timestamp) {
      if (!timestamp) throw new Error('Did not recieve a timestamp')

      let t = moment(timestamp).tz(this.timezone),
        shortenedTz = moment()
          .tz(this.timezone || Intl.DateTimeFormat().resolvedOptions().timeZone)
          .zoneAbbr()

      let timeObj = t ? t : moment(timestamp)

      let formatted = timeObj.calendar(null, {
        sameDay: 'h:mma',
        sameElse: 'MMMM D, YYYY [at] h:mma'
      })
      return `${formatted} ${shortenedTz}`
    },
    getTimeOverdue(time) {
      let now, start
      if (this.timezone) {
        now = new moment().tz(this.timezone)
        start = moment(time).tz(this.timezone)
      } else {
        now = new moment()
        start = moment(time)
      }
      let diff = moment.duration(now.diff(start))

      return diff
    }
  },
  apollo: {
    upcoming: {
      query: require('@/graphql/Flow/upcoming-flow-runs.gql'),
      variables() {
        let variables = {}

        if (this.aggregate) {
          variables.flow_group_id = this.flow.flow_group_id
        } else {
          variables.flow_id = this.flow.id
        }

        return variables
      },
      loadingKey: 'loading',
      update({ flow_run }) {
        if (!flow_run) return
        return flow_run
      }
    }
  }
}
</script>

<template>
  <v-card
    class="py-2"
    tile
    :style="{
      height: fullHeight ? '100%' : 'auto'
    }"
  >
    <v-system-bar
      :color="
        flow.archived
          ? 'grey'
          : loading > 0
          ? 'secondaryGray'
          : lateRuns.length > 0
          ? 'deepRed'
          : 'Success'
      "
      :height="5"
      absolute
    >
    </v-system-bar>

    <CardTitle :title="title" :icon="titleIcon" :icon-color="titleIconColor">
      <v-row slot="title" no-gutters class="d-flex align-center">
        <v-col cols="8">
          <div>
            <div
              v-if="loading > 0 || (tab === 'late' && isClearingLateRuns)"
              style="
                display: inline-block;
                height: 20px;
                overflow: hidden;
                width: 20px;"
            >
              <v-skeleton-loader type="avatar" tile></v-skeleton-loader>
            </div>
            {{ title }}
          </div>
          <ConcurrencyInfo
            v-if="isCloud && tab === 'late'"
            class="caption position-absolute"
            style="bottom: 0;"
          />
        </v-col>
      </v-row>
      <div
        v-if="!flow.archived"
        slot="action"
        class="d-flex flex-column align-end"
      >
        <v-btn
          depressed
          small
          tile
          icon
          class="button-transition w-100 d-flex justify-end"
          :color="tab == 'upcoming' ? 'primary' : ''"
          :style="{
            'border-right': `3px solid ${
              tab == 'upcoming' ? 'var(--v-primary-base)' : 'transparent'
            }`,
            'box-sizing': 'content-box',
            'min-width': '100px'
          }"
          @click="tab = 'upcoming'"
        >
          Upcoming
          <v-icon small>access_time</v-icon>
        </v-btn>

        <v-btn
          depressed
          small
          tile
          icon
          class="button-transition w-100 d-flex justify-end"
          :color="tab == 'late' ? 'primary' : ''"
          :style="{
            'border-right': `3px solid ${
              tab == 'late'
                ? lateRuns.length > 0
                  ? 'var(--v-deepRed-base)'
                  : 'var(--v-primary-base)'
                : 'transparent'
            }`,
            'box-sizing': 'content-box',
            'min-width': '100px'
          }"
          @click="tab = 'late'"
        >
          <v-icon v-if="lateRuns.length > 0" small color="deepRed">
            warning
          </v-icon>
          Late
          <v-icon small>timelapse</v-icon>
        </v-btn>
      </div>
    </CardTitle>

    <v-card-text v-if="tab == 'upcoming'" class="pa-0 card-content">
      <v-skeleton-loader v-if="loading > 0" type="list-item-three-line">
      </v-skeleton-loader>

      <v-list-item v-else-if="loading === 0 && upcomingRuns.length === 0" dense>
        <v-list-item-avatar class="mr-0">
          <v-icon :color="flow.archived ? '' : 'Success'">
            {{ flow.archived ? 'archive' : 'check' }}
          </v-icon>
        </v-list-item-avatar>
        <v-list-item-content class="my-0 py-0">
          <div
            class="subtitle-1 font-weight-light"
            style="line-height: 1.25rem;"
          >
            {{
              flow.archived
                ? "This is an archived flow version so it won't have upcoming runs."
                : 'No upcoming runs.'
            }}
          </div>
        </v-list-item-content>
      </v-list-item>

      <v-list v-else dense>
        <v-lazy
          v-for="item in upcomingRuns"
          :key="item.id"
          :options="{
            threshold: 0.75
          }"
          min-height="40px"
          transition="fade"
        >
          <v-list-item dense :disabled="setToRun.includes(item.id)">
            <v-list-item-content>
              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <span class="caption mb-0" v-on="on">
                    Scheduled for {{ formatTime(item.scheduled_start_time) }}
                  </span>
                </template>
                <span>
                  {{ formatDate(item.scheduled_start_time) }}
                </span>
              </v-tooltip>

              <v-list-item-subtitle class="font-weight-light">
                <router-link
                  :to="{ name: 'flow-run', params: { id: item.id } }"
                >
                  {{ item.name }}
                </router-link>
              </v-list-item-subtitle>
            </v-list-item-content>

            <v-list-item-action tile min-width="5" class="body-2">
              <v-tooltip top>
                <template v-slot:activator="{ on }">
                  <v-btn
                    text
                    x-small
                    aria-label="Run Now"
                    :disabled="setToRun.includes(item.id)"
                    color="primary"
                    class="vertical-button"
                    v-on="on"
                    @click="runFlowNow(item.id, item.version, item.name)"
                  >
                    <v-icon x-small dense color="primary"> fa-rocket</v-icon>
                  </v-btn>
                </template>
                <span> Run {{ item.name }} now </span>
              </v-tooltip>
            </v-list-item-action>
          </v-list-item>
        </v-lazy>
      </v-list>

      <Alert
        v-model="showAlert"
        :type="alertType"
        :message="alertMessage"
        :alert-link="alertLink"
        :timeout="12000"
      >
      </Alert>

      <div v-if="upcomingRuns.length > 3" class="pa-0 card-footer"> </div>
    </v-card-text>

    <v-card-text v-if="tab == 'late'" class="pa-0 card-content">
      <v-skeleton-loader
        v-if="loading > 0 || isClearingLateRuns"
        type="list-item-three-line"
      >
      </v-skeleton-loader>

      <v-list-item v-else-if="loading === 0 && lateRuns.length === 0" dense>
        <v-list-item-avatar class="mr-0">
          <v-icon class="green--text">check</v-icon>
        </v-list-item-avatar>
        <v-list-item-content class="my-0 py-0">
          <div
            class="subtitle-1 font-weight-light"
            style="line-height: 1.25rem;"
          >
            Everything is running on schedule!
          </div>
        </v-list-item-content>
      </v-list-item>

      <v-list v-else dense>
        <v-lazy
          v-for="item in lateRuns"
          :key="item.id"
          :options="{
            threshold: 0.75
          }"
          min-height="40px"
          transition="fade"
        >
          <v-list-item
            dense
            two-line
            :to="{ name: 'flow-run', params: { id: item.id } }"
          >
            <v-list-item-content>
              <span class="caption mb-0">
                Scheduled for {{ formatTime(item.scheduled_start_time) }}
              </span>
              <v-list-item-title class="body-2">
                <router-link
                  :to="{ name: 'flow-run', params: { id: item.id } }"
                >
                  {{ item.name }}
                </router-link>
              </v-list-item-title>
              <v-list-item-subtitle class="caption">
                <DurationSpan :start-time="item.scheduled_start_time" />
                behind schedule
              </v-list-item-subtitle>
            </v-list-item-content>

            <v-list-item-avatar class="body-2">
              <v-icon class="grey--text">arrow_right</v-icon>
            </v-list-item-avatar>
          </v-list-item>
        </v-lazy>

        <v-btn
          text
          color="deepRed"
          small
          :loading="isClearingLateRuns"
          class="position-absolute"
          :style="{ bottom: '12px', right: '4px' }"
          tile
          @click="handleOpenDialog"
        >
          Clear
        </v-btn>

        <v-dialog v-model="showClearLateRunsDialog" max-width="540">
          <v-card flat>
            <v-card-title class="title word-break-normal">
              Are you sure you want to clear all late runs for this flow?
            </v-card-title>

            <v-card-text>
              This will set all late flow runs in a
              <strong>Cancelled</strong> state.
            </v-card-text>

            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn text tile @click="showClearLateRunsDialog = false">
                Cancel
              </v-btn>
              <v-btn text color="deepRed" tile @click="clearLateRuns">
                Confirm
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
      </v-list>

      <div v-if="lateRuns.length > 3" class="pa-0 card-footer"> </div>
    </v-card-text>

    <Alert
      v-model="clearLateRunsError"
      type="error"
      message="Something went wrong while trying to clear your late flow runs. Please try again later."
    ></Alert>
  </v-card>
</template>

<style lang="scss" scoped>
a {
  text-decoration: none !important;
}

.w-100 {
  width: 100% !important;
}

.button-transition {
  transition: border-right 150ms linear;
}

.card-content {
  max-height: 254px;
  overflow-y: scroll;
}

.card-footer {
  background-image: linear-gradient(transparent, 60%, rgba(0, 0, 0, 0.1));
  bottom: 6px;
  height: 6px !important;
  pointer-events: none;
  position: absolute;
  width: 100%;
}
</style>
