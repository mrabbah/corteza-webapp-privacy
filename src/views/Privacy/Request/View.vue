<template>
  <b-container
    class="d-flex flex-column p-3"
  >
    <request-viewer
      v-if="request"
      :request="request"
      class="mb-3"
    />

    <request-comments
      v-if="request"
      :comments="comments"
      :sort="sort"
      @sort="sort = $event"
      @submit="submitComment"
    />

    <portal to="editor-toolbar">
      <editor-toolbar
        :processing="processing"
        :back-link="{ name: 'request.list' }"
        :delete-show="isDC"
        :delete-disabled="!request || !isPending"
        :delete-label="$t('reject')"
        @delete="handleRequest('rejected')"
      >
        <template #right>
          <c-input-confirm
            v-if="!isDC"
            :borderless="false"
            :disabled="!request || !isPending"
            variant="light"
            size="lg"
            size-confirm="lg"
            @confirmed="handleRequest('canceled')"
          >
            Cancel Request
          </c-input-confirm>

          <c-input-confirm
            v-else
            :borderless="false"
            :disabled="!request || !isPending"
            variant="primary"
            size="lg"
            size-confirm="lg"
            class="ml-2"
            @confirmed="handleRequest('approved')"
          >
            {{ $t('approve') }}
          </c-input-confirm>
        </template>
      </editor-toolbar>
    </portal>
  </b-container>
</template>

<script>
import EditorToolbar from 'corteza-webapp-privacy/src/components/Common/EditorToolbar'
import RequestViewer from 'corteza-webapp-privacy/src/components/Requests/Viewer'
import RequestComments from 'corteza-webapp-privacy/src/components/Requests/Comments'

export default {
  name: 'RequestView',

  i18nOptions: {
    namespaces: 'request',
    keyPrefix: 'view',
  },

  components: {
    EditorToolbar,
    RequestViewer,
    RequestComments,
  },

  props: {
    requestID: {
      type: String,
      required: false,
      default: '',
    },
  },

  data () {
    return {
      processing: false,

      isDC: null,

      sort: 'createdAt DESC',

      request: undefined,
      comments: [],
    }
  },

  computed: {
    isPending () {
      return this.request.status === 'pending'
    },
  },

  watch: {
    requestID: {
      immediate: true,
      handler () {
        if (this.requestID) {
          this.fetchRequest()
          this.fetchComments()
        }
      },
    },

    sort: {
      handler () {
        if (this.requestID) {
          this.fetchComments()
        }
      },
    },
  },

  created () {
    this.checkIsDC()
  },

  methods: {
    checkIsDC () {
      this.$SystemAPI.roleList({ query: 'data-privacy-officer', memberID: this.$auth.user.userID })
        .then(({ set = [] }) => {
          this.isDC = !!set.length
        })
    },

    fetchRequest (requestID = this.requestID) {
      this.processing = true

      return this.$SystemAPI.dataPrivacyRequestRead({ requestID })
        .then(request => {
          this.request = request
        })
        .catch(this.toastErrorHandler(this.$t('notification:list.load.error')))
        .finally(() => {
          this.processing = false
        })
    },

    fetchComments (requestID = this.requestID) {
      this.processing = true

      return this.$SystemAPI.dataPrivacyRequestCommentList({ requestID, sort: this.sort })
        .then(({ set }) => {
          this.comments = set
        })
        .catch(this.toastErrorHandler(this.$t('notification:list.load.error')))
        .finally(() => {
          this.processing = false
        })
    },

    handleRequest (status) {
      this.processing = true

      this.$SystemAPI.dataPrivacyRequestUpdateStatus({ requestID: this.requestID, status })
        .then(() => {
          this.$router.push({ name: 'request.list' })
        })
        .finally(() => {
          this.processing = true
        })
    },

    submitComment (comment) {
      this.processing = true

      this.$SystemAPI.dataPrivacyRequestCommentCreate({ requestID: this.requestID, comment })
        .then(() => {
          this.fetchComments()
        })
        .finally(() => {
          this.processing = true
        })
    },
  },
}
</script>
