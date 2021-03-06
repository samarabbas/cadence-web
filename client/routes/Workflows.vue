<template>
  <section :class="{ workflows: true, loading }">
    <header class="filters">
      <div class="field workflow-id">
        <input type="search" class="workflow-id"
          placeholder=" "
          name="workflowId"
          v-bind:value="$route.query.workflowId"
          @input="setWorkflowFilter" />
        <label for="workflowId">Workflow ID</label>
      </div>
      <div class="field workflow-name">
        <input type="search" class="workflow-name"
          placeholder=" "
          name="workflowName"
          v-bind:value="$route.query.workflowName"
          @input="setWorkflowFilter" />
        <label for="workflowName">Workflow Name</label>
      </div>
      <date-range-picker
        :date-range="range"
        @change="setRange"
      />
      <v-select
        class="status"
        :value="status"
        :options="statuses"
        :on-change="setStatus"
        :searchable="false"
      />
    </header>
    <section class="results"
      v-infinite-scroll="nextPage"
      infinite-scroll-disabled="disableInfiniteScroll"
      infinite-scroll-distance="20"
      infinite-scroll-immediate-check="false"
    >
      <table v-show="showTable">
        <thead>
          <th>Workflow ID</th>
          <th>Run ID</th>
          <th>Name</th>
          <th>Status</th>
          <th>Start Time</th>
          <th>End Time</th>
        </thead>
        <tbody>
          <tr v-for="wf in results" :key="wf.runId">
            <td>{{wf.workflowId}}</td>
            <td><router-link :to="{ name: 'execution/summary', params: { runId: wf.runId, workflowId: wf.workflowId }}">{{wf.runId}}</router-link></td>
            <td>{{wf.workflowName}}</td>
            <td :class="wf.status">{{wf.status}}</td>
            <td>{{wf.startTime}}</td>
            <td>{{wf.endTime}}</td>
          </tr>
        </tbody>
      </table>
    </section>
    <span class="error" v-if="error">{{error}}</span>
    <span class="no-results" v-if="showNoResults">No Results</span>
  </section>
</template>

<script>
import moment from 'moment'
import pagedGrid from '../paged-grid'
import debounce from 'lodash-es/debounce'

export default pagedGrid({
  data() {
    return {
      loading: false,
      results: [],
      error: undefined,
      nextPageToken: undefined,
      statuses: [
        { value: 'OPEN', label: 'Open' },
        { value: 'CLOSED', label: 'Closed' },
        { value: 'COMPLETED', label: 'Completed' },
        { value: 'FAILED', label: 'Failed' },
        { value: 'CANCELED', label: 'Cancelled' },
        { value: 'TERMINATED', label: 'Terminated' },
        { value: 'CONTINUED_AS_NEW', label: 'Continued As New' },
        { value: 'TIMED_OUT', label: 'Timed Out'}
      ]
    }
  },
  created() {
    var q = this.$route.query || {}
    if (!q.range || !/^last-\d{1,2}-(hour|day|month)s?$/.test(q.range)) {
      this.setRange(localStorage.getItem(`${this.$route.params.domain}:workflows-time-range`) || 'last-30-days')
    }
    this.$watch('queryOnChange', () => {}, { immediate: true })
  },
  computed: {
    status() {
      if (!this.$route.query || !this.$route.query.status) {
        return this.statuses[0]
      }
      return this.statuses.find(s => s.value === this.$route.query.status)
    },
    range() {
      var q = this.$route.query || {}
      return q.startTime && q.endTime ? {
        startTime: moment(q.startTime),
        endTime: moment(q.endTime)
      } : q.range
    },
    criteria() {
      var
        domain = this.$route.params.domain,
        q = this.$route.query,
        { startTime, endTime } = q

      if (q.range && typeof q.range === 'string') {
        let [,count,unit] = q.range.split('-')
        startTime = moment().subtract(count, unit).startOf(unit).toISOString()
        endTime = moment().endOf(unit).toISOString()
      }

      this.nextPageToken = undefined
      return {
        domain,
        startTime,
        endTime,
        status: q.status,
        workflowId: q.workflowId,
        workflowName: q.workflowName
      }
    },
    queryOnChange() {
      var
        q = Object.assign({}, this.criteria),
        domain = q.domain,
        state = (!q.status || q.status === 'OPEN') ? 'open' : 'closed'

      if (!q.startTime || !q.endTime || !q.status) return
      if (['OPEN', 'CLOSED'].includes(q.status)) {
        delete q.status
      }
      delete q.domain
      q.nextPageToken = this.nextPageToken

      return this.fetch(`/api/domain/${domain}/workflows/${state}`, q)
    }
  },
  methods: {
    fetch: debounce(function(url, query) {
      this.loading = true
      this.error = undefined

      return this.$http(url, { query })
      .then(res => {
        this.npt = res.nextPageToken
        this.loading = false
        var formattedResults = res.executions.map(data => ({
          workflowId: data.execution.workflowId,
          runId: data.execution.runId,
          workflowName: data.type.name,
          startTime: moment(data.startTime).format('lll'),
          endTime: data.closeTime ? moment(data.closeTime).format('lll') : '',
          status: (data.closeStatus || 'open').toLowerCase(),
        }))
        return this.results = query.nextPageToken ? this.results.concat(formattedResults) : formattedResults
      }).catch(e => {
        this.npt = undefined
        this.loading = false
        this.error = (e.json && e.json.message) || e.status || e.message
        return []
      })
    }, typeof Mocha === 'undefined' ? 200 : 60, { maxWait: 1000 }),
    setWorkflowFilter(e) {
      var target = e.target || e.testTarget // test hook since Event.target is readOnly and unsettable
      this.$router.replaceQueryParam(target.getAttribute('name'), target.value)
    },
    setStatus(status) {
      if (status) {
        this.$router.replace({
          query: Object.assign({}, this.$route.query, { status: status.value })
        })
      }
    },
    setRange(range) {
      if (range) {
        var query = Object.assign({}, this.$route.query)
        if (typeof range === 'string') {
          query.range = range
          delete query.startTime
          delete query.endTime
          localStorage.setItem(`${this.$route.params.domain}:workflows-time-range`, range)
        } else {
          query.startTime = range.startTime.toISOString()
          query.endTime = range.endTime.toISOString()
          delete query.range
        }
        this.$router.replace({ query })
      }
    }
  }
})
</script>

<style lang="stylus">
@require "../styles/definitions.styl"

section.workflows
  .filters
    flex-wrap wrap
    > .field
      flex 1 1 auto
    .date-range-picker
      flex 1 1 auto
    .v-select
      flex 1 1 240px

  paged-grid()

  &.loading section.results table
    opacity 0.7

  table
    td:nth-child(4)
      text-transform capitalize
      &.completed
        color uber-green
      &.failed
        color uber-orange
      &.open
        color uber-blue-120
    td:nth-child(5), td:nth-child(6)
      one-liner-ellipsis()
</style>