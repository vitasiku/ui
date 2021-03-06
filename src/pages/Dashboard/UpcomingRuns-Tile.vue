<script>
// import moment from 'moment-timezone'
import { mapGetters } from 'vuex'
import CardTitle from '@/components/Card-Title'
import ConcurrencyInfo from '@/components/ConcurrencyInfo'
import DurationSpan from '@/components/DurationSpan'
import ExternalLink from '@/components/ExternalLink'
import LabelWarning from '@/components/LabelWarning'
import { cancelLateRunsMixin } from '@/mixins/cancelLateRunsMixin'
import { runFlowNowMixin } from '@/mixins/runFlowNow'
import { formatTime } from '@/mixins/formatTimeMixin'

export default {
  components: {
    CardTitle,
    ConcurrencyInfo,
    DurationSpan,
    ExternalLink,
    LabelWarning
  },
  mixins: [cancelLateRunsMixin, runFlowNowMixin, formatTime],
  props: {
    projectId: {
      required: false,
      type: String,
      default: () => null
    },
    fullHeight: {
      required: false,
      type: Boolean,
      default: () => false
    }
  },
  data() {
    return {
      loading: 0,
      tab: 'upcoming'
    }
  },
  computed: {
    ...mapGetters('api', ['isCloud']),
    ...mapGetters('tenant', ['tenant']),
    ...mapGetters('user', ['timezone']),
    lateRuns() {
      if (!this.upcoming) return null
      return this.upcoming.filter(run => {
        return this.getTimeOverdue(run.scheduled_start_time) > 20000
      })
    },
    upcomingRuns() {
      if (!this.upcoming) return null
      return this.upcoming.filter(run => {
        return this.getTimeOverdue(run.scheduled_start_time) <= 20000
      })
    },
    title() {
      let title = ''

      if (this.tab == 'upcoming') {
        title =
          this.loading > 0
            ? 'Upcoming Runs'
            : `${this.upcomingRuns?.length || 0} Upcoming Runs`
      }

      if (this.tab == 'late') {
        title =
          this.loading > 0 || this.isClearingLateRuns
            ? 'Late Runs'
            : `${this.lateRuns?.length || 0} Late Runs`
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
      return this.loading > 0
        ? 'grey'
        : this.tab == 'upcoming'
        ? 'primary'
        : this.lateRuns?.length > 0
        ? 'deepRed'
        : 'Success'
    }
  },
  watch: {
    upcoming(val) {
      if (!val) return
      if (this.lateRuns?.length > 0) {
        this.tab = 'late'
      }
    },
    tenant(val) {
      this.projects = []

      if (val) {
        this.loading = 1
        setTimeout(async () => {
          await this.$apollo.queries.upcoming.refetch(), (this.loading = 0)
        }, 1000)
      }
    }
  },
  methods: {
    getTimeOverdue(time) {
      return new Date() - new Date(time)
    }
  },
  apollo: {
    upcoming: {
      query: require('@/graphql/Dashboard/upcoming-flow-runs.gql'),
      variables() {
        return {
          projectId: this.projectId ? this.projectId : null
        }
      },
      loadingKey: 'loading',
      pollInterval: 10000,
      update: data => data?.flow_run
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
        loading > 0
          ? 'secondaryGray'
          : lateRuns && lateRuns.length > 0
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
            style="bottom: 2px;"
          />
        </v-col>
      </v-row>

      <div slot="action" class="d-flex align-end flex-column">
        <v-btn
          depressed
          small
          tile
          icon
          class="button-transition w-100 d-flex justify-end"
          :color="tab == 'upcoming' ? 'primary' : ''"
          :style="{
            'border-right': `3px solid ${
              tab == 'upcoming' ? 'var(--v-primary-base)' : '#fff'
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
                ? lateRuns && lateRuns.length > 0
                  ? 'var(--v-deepRed-base)'
                  : 'var(--v-primary-base)'
                : '#fff'
            }`,
            'box-sizing': 'content-box',
            'min-width': '100px'
          }"
          @click="tab = 'late'"
        >
          <v-icon v-if="lateRuns && lateRuns.length > 0" small color="deepRed">
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

      <v-list-item
        v-else-if="loading === 0 && upcomingRuns && upcomingRuns.length === 0"
        dense
      >
        <v-list-item-avatar class="mr-0">
          <v-icon class="green--text">check</v-icon>
        </v-list-item-avatar>
        <v-list-item-content class="my-0 py-0">
          <div
            class="subtitle-1 font-weight-light"
            style="line-height: 1.25rem;"
          >
            No upcoming runs.
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
          min-height="44"
          transition="fade-transition"
        >
          <v-list-item dense :disabled="setToRun.includes(item.id)">
            <v-list-item-content>
              <span class="caption mb-0 ml-n1 d-flex align-end">
                <LabelWarning
                  :flow="item.flow"
                  :flow-group="item.flow.flow_group"
                />
                <span class="ml-1">
                  Scheduled for
                  {{ formatDateTime(item.scheduled_start_time) }}
                </span>
              </span>
              <v-list-item-subtitle class="font-weight-light">
                <router-link
                  :to="{ name: 'flow', params: { id: item.flow.id } }"
                >
                  {{ item.flow.name }}
                </router-link>
                <span class="font-weight-bold">
                  <v-icon style="font-size: 12px;">
                    chevron_right
                  </v-icon>
                </span>
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
                    <v-icon small dense color="primary"> fa-rocket</v-icon>
                  </v-btn>
                </template>
                <span> Run {{ item.name }} now </span>
              </v-tooltip>
            </v-list-item-action>
          </v-list-item>
        </v-lazy>
      </v-list>

      <div
        v-if="upcomingRuns && upcomingRuns.length > 3"
        class="pa-0 card-footer"
      >
      </div>
    </v-card-text>

    <v-card-text v-if="tab == 'late'" class="pa-0 card-content">
      <v-skeleton-loader
        v-if="loading > 0 || isClearingLateRuns"
        type="list-item-three-line"
      >
      </v-skeleton-loader>

      <v-list-item
        v-else-if="loading === 0 && lateRuns && lateRuns.length === 0"
        dense
      >
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
            three-line
            :to="{ name: 'flow-run', params: { id: item.id } }"
          >
            <v-list-item-content>
              <span class="caption mb-0 ml-n1 d-flex align-end">
                <LabelWarning
                  :flow="item.flow"
                  :flow-group="item.flow.flow_group"
                />
                <span class="ml-1">
                  Scheduled for
                  {{ formatDateTime(item.scheduled_start_time) }}
                </span>
              </span>

              <v-list-item-title class="body-2">
                <router-link
                  :to="{ name: 'flow', params: { id: item.flow.id } }"
                >
                  {{ item.flow.name }}
                </router-link>
                <span class="font-weight-bold">
                  <v-icon style="font-size: 12px;">
                    chevron_right
                  </v-icon>
                </span>
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
          class="position-absolute"
          :style="{ bottom: '12px', right: '4px' }"
          tile
          :loading="isClearingLateRuns"
          @click="handleOpenDialog"
        >
          Clear
        </v-btn>

        <v-dialog v-model="showClearLateRunsDialog" max-width="480">
          <v-card flat>
            <v-card-title class="title word-break-normal">
              Are you sure you want to clear all late runs?
            </v-card-title>

            <v-card-text v-if="scheduleIds.length > 1">
              This will toggle the schedule for all flows with late runs. If you
              did not set a
              <ExternalLink
                href="https://docs.prefect.io/core/concepts/schedules.html#schedules"
                >start time</ExternalLink
              >
              for your schedule,
              <strong
                >it may affect the scheduled start times for your upcoming
                runs.</strong
              >
            </v-card-text>
            <v-card-text v-else>
              This will set your late flow runs into a
              <strong>cancelled</strong> state.
            </v-card-text>

            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn text tile @click="showClearLateRunsDialog = false">
                Cancel
              </v-btn>
              <v-btn dark color="deepRed" depressed @click="clearLateRuns">
                Confirm
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
      </v-list>

      <div v-if="lateRuns && lateRuns.length > 3" class="pa-0 card-footer">
      </div>
    </v-card-text>
  </v-card>
</template>

<style lang="scss" scoped>
a {
  text-decoration: none !important;
}

.button-transition {
  transition: border-right 150ms linear;
}

.w-100 {
  width: 100% !important;
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
