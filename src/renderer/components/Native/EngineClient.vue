<template>
  <div v-if="false"></div>
</template>

<script>
  import is from 'electron-is'
  import { mapState } from 'vuex'
  import api from '@/api'
  import {
    getTaskFullPath,
    showItemInFolder
  } from '@/utils/native'
  import { checkTaskIsBT, getTaskName, getFileNameFromUrl } from '@shared/utils'
  import { ENGINE_STATUS } from '@shared/constants'

  export default {
    name: 'mo-engine-client',
    computed: {
      isRenderer: () => is.renderer(),
      ...mapState('app', {
        uploadSpeed: state => state.stat.uploadSpeed,
        downloadSpeed: state => state.stat.downloadSpeed,
        speed: state => state.stat.uploadSpeed + state.stat.downloadSpeed,
        interval: state => state.interval,
        downloading: state => state.stat.numActive > 0,
        progress: state => state.progress
      }),
      ...mapState('task', {
        messages: state => state.messages,
        seedingList: state => state.seedingList,
        taskDetailVisible: state => state.taskDetailVisible,
        enabledFetchPeers: state => state.enabledFetchPeers,
        currentTaskGid: state => state.currentTaskGid,
        currentTaskItem: state => state.currentTaskItem
      }),
      ...mapState('preference', {
        taskStartNotification: state => state.config.taskStartNotification,
        taskNotificationSilent: state => state.config.taskNotificationSilent,
        taskNotification: state => state.config.taskNotification
      }),
      currentTaskIsBT () {
        return checkTaskIsBT(this.currentTaskItem)
      }
    },
    watch: {
      speed (val) {
        const { uploadSpeed, downloadSpeed } = this
        this.$electron.ipcRenderer.send('event', 'speed-change', {
          uploadSpeed,
          downloadSpeed
        })
      },
      downloading (val, oldVal) {
        if (val !== oldVal && this.isRenderer) {
          this.$electron.ipcRenderer.send('event', 'download-status-change', val)
        }
      },
      progress (val) {
        this.$electron.ipcRenderer.send('event', 'progress-change', val)
      }
    },
    methods: {
      async fetchTaskItem ({ gid }) {
        return api.fetchTaskItem({ gid })
          .catch((e) => {
            console.warn(`fetchTaskItem fail: ${e.message}`)
          })
      },
      onDownloadStart (event) {
        this.$store.dispatch('task/fetchList')
        this.$store.dispatch('app/resetInterval')
        this.$store.dispatch('task/saveSession')
        this.startPolling()
        const [{ gid }] = event
        const { seedingList } = this
        if (seedingList.includes(gid)) {
          return
        }

        this.fetchTaskItem({ gid })
          .then((task) => {
            const { dir } = task
            this.$store.dispatch('preference/recordHistoryDirectory', dir)
            this.$store.dispatch('task/recordDownloadEventTime', { event: 'start', task: task, timestamp: Date.now() })
            const taskName = getFileNameFromUrl(task)
            const message = this.$t('task.download-start-message', { taskName })
            this.$msg.info(message)
            this.showTaskStartNotify(task)
          })
      },
      onDownloadPause (event) {
        this.stopPolling()
        const [{ gid }] = event
        const { seedingList } = this
        if (seedingList.includes(gid)) {
          return
        }

        this.fetchTaskItem({ gid })
          .then((task) => {
            this.$store.dispatch('task/recordDownloadEventTime', { event: 'pause', task: task, timestamp: Date.now() })
            const taskName = getTaskName(task)
            const message = this.$t('task.download-pause-message', { taskName })
            this.$msg.info(message)
          })
      },
      onDownloadStop (event) {
        this.stopPolling()
        const [{ gid }] = event
        this.fetchTaskItem({ gid })
          .then((task) => {
            const taskName = getTaskName(task)
            const message = this.$t('task.download-stop-message', { taskName })
            this.$msg.info(message)
          })
      },
      onDownloadError (event) {
        this.stopPolling()
        const [{ gid }] = event
        this.fetchTaskItem({ gid })
          .then((task) => {
            const taskName = getTaskName(task)
            const { errorCode, errorMessage } = task
            console.error(`[Motrix] download error gid: ${gid}, #${errorCode}, ${errorMessage}`)
            const message = this.$t('task.download-error-message', { taskName })
            const link = `<a target="_blank" href="https://github.com/agalwood/Motrix/wiki/Error#${errorCode}" rel="noopener noreferrer">${errorCode}</a>`
            this.$msg({
              type: 'error',
              showClose: true,
              duration: 5000,
              dangerouslyUseHTMLString: true,
              message: `${message} ${link}`
            })
          })
      },
      onDownloadComplete (event) {
        this.stopPolling()
        const [{ gid }] = event
        this.$store.dispatch('task/removeFromSeedingList', gid)

        this.fetchTaskItem({ gid })
          .then((task) => {
            this.handleDownloadComplete(task, false)
          })
      },
      onBtDownloadComplete (event) {
        this.stopPolling()
        const [{ gid }] = event
        const { seedingList } = this
        if (seedingList.includes(gid)) {
          return
        }

        this.$store.dispatch('task/addToSeedingList', gid)

        this.fetchTaskItem({ gid })
          .then((task) => {
            this.handleDownloadComplete(task, true)
          })
      },
      onEngineStatusChanged (status) {
        const lastStatus = this.$store.state.app.engineStatus
        if (lastStatus === status) {
          return
        }

        const connected = status === ENGINE_STATUS.CONNECTED
        const connecting = status === ENGINE_STATUS.CONNECTING

        const statusTypeMap = {
          [ENGINE_STATUS.CONNECTED]: 'success',
          [ENGINE_STATUS.CONNECTING]: 'info',
          [ENGINE_STATUS.DISCONNECTED]: 'error'
        }

        const statusStringMap = {
          [ENGINE_STATUS.CONNECTED]: this.$t('app.engine-connected-message'),
          [ENGINE_STATUS.CONNECTING]: this.$t('app.engine-connecting-message'),
          [ENGINE_STATUS.DISCONNECTED]: this.$t('app.engine-disconnected-message')
        }

        if (this.lastStatusMsg) {
          this.lastStatusMsg.close()
        }

        if (connecting) {
          this.closeClientTimer = setTimeout(() => {
            this.$store.dispatch('app/changeClientStatus', 'close')
          }, 10000)
        } else {
          clearTimeout(this.closeClientTimer)
          this.closeClientTimer = null
        }

        this.lastStatusMsg = this.$msg({
          type: statusTypeMap[status],
          message: statusStringMap[status],
          showClose: false,
          duration: connected ? 2000 : 0
        })

        console.log(`[Motrix] engine status changed status: ${status} lastStatus: ${lastStatus}`)

        this.polling()
        this.$store.dispatch('app/updateEngineStatus', status)
      },
      handleDownloadComplete (task, isBT) {
        this.$store.dispatch('task/saveSession')
        this.$store.dispatch('task/recordDownloadEventTime', { event: 'complete', task: task, timestamp: Date.now() })

        const path = getTaskFullPath(task)
        this.showTaskCompleteNotify(task, isBT, path)
        this.$electron.ipcRenderer.send('event', 'task-download-complete', task, path)
      },
      showTaskStartNotify (task) {
        if (!this.taskStartNotification) {
          return
        }

        const taskName = getFileNameFromUrl(task)

        /* eslint-disable no-new */
        const notify = new Notification(this.$t('task.download-start-message', { taskName: ' ' }), {
          body: taskName,
          silent: this.taskNotificationSilent
        })
        notify.onclick = () => {
          this.$electron.ipcRenderer.send('command', 'application:show', 'index')
        }
      },
      showTaskCompleteNotify (task, isBT, path) {
        const taskName = getTaskName(task)
        const message = isBT
          ? this.$t('task.bt-download-complete-message', { taskName })
          : this.$t('task.download-complete-message', { taskName })
        const tips = isBT
          ? '\n' + this.$t('task.bt-download-complete-tips')
          : ''

        this.$msg.success(`${message}${tips}`)

        if (!this.taskNotification) {
          return
        }

        const notifyMessage = isBT
          ? this.$t('task.bt-download-complete-notify')
          : this.$t('task.download-complete-notify')

        /* eslint-disable no-new */
        const notify = new Notification(notifyMessage, {
          body: `${taskName}${tips}`,
          silent: this.taskNotificationSilent
        })
        notify.onclick = () => {
          showItemInFolder(path, {
            errorMsg: this.$t('task.file-not-exist')
          })
        }
      },
      showTaskErrorNotify (task) {
        const taskName = getTaskName(task)

        const message = this.$t('task.download-fail-message', { taskName })
        this.$msg.success(message)

        if (!this.taskNotification) {
          return
        }

        /* eslint-disable no-new */
        const notify = new Notification(this.$t('task.download-fail-notify'), {
          body: taskName,
          silent: this.taskNotificationSilent
        })
        notify.onclick = () => {
          this.$electron.ipcRenderer.send('command', 'application:show', 'index')
        }
      },
      bindEngineEvents () {
        api.client.on('onDownloadStart', this.onDownloadStart)
        api.client.on('onDownloadPause', this.onDownloadPause)
        api.client.on('onDownloadStop', this.onDownloadStop)
        api.client.on('onDownloadComplete', this.onDownloadComplete)
        api.client.on('onDownloadError', this.onDownloadError)
        api.client.on('onBtDownloadComplete', this.onBtDownloadComplete)
        api.client.on('onEngineStatusChanged', this.onEngineStatusChanged)
      },
      unbindEngineEvents () {
        api.client.removeListener('onDownloadStart', this.onDownloadStart)
        api.client.removeListener('onDownloadPause', this.onDownloadPause)
        api.client.removeListener('onDownloadStop', this.onDownloadStop)
        api.client.removeListener('onDownloadComplete', this.onDownloadComplete)
        api.client.removeListener('onDownloadError', this.onDownloadError)
        api.client.removeListener('onBtDownloadComplete', this.onBtDownloadComplete)
        api.client.removeListener('onEngineStatusChanged', this.onEngineStatusChanged)
      },
      startPolling () {
        this.timer = setTimeout(() => {
          this.polling()
          this.startPolling()
        }, this.interval)
      },
      polling () {
        this.$store.dispatch('app/fetchGlobalStat')
        this.$store.dispatch('app/fetchProgress')
        this.$store.dispatch('task/fetchList')

        if (this.taskDetailVisible && this.currentTaskGid) {
          if (this.currentTaskIsBT && this.enabledFetchPeers) {
            this.$store.dispatch('task/fetchItemWithPeers', this.currentTaskGid)
          } else {
            this.$store.dispatch('task/fetchItem', this.currentTaskGid)
          }
        }
      },
      stopPolling () {
        clearTimeout(this.timer)
        this.timer = null
        this.polling()
      }
    },
    created () {
      this.bindEngineEvents()
    },
    mounted () {
      setTimeout(() => {
        this.$store.dispatch('app/fetchEngineInfo')
        this.$store.dispatch('app/fetchEngineOptions')
      }, 100)
    },
    destroyed () {
      this.$store.dispatch('task/saveSession')

      this.unbindEngineEvents()
    }
  }
</script>
