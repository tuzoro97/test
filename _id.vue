<template>
  <div class="px-4 suspect-detail">
    <my-breadcrumbs v-show="!expandMeetingDetail" text="Quản lý cuộc họp | ">
      <template v-slot:append>
        <span class="name-suspect">&nbsp;{{ infoObject.name }}</span>
      </template>
    </my-breadcrumbs>
    <span
      v-if="!expandMeetingDetail && !panel"
      @click="panel = true"
      class="more-info-meeting"
    >
      Thông tin về cuộc họp
      <v-hover v-slot:default="{ hover }">
        <img
          :src="
            hover
              ? require('~/assets/images/icon/expan-more-info_2__active.svg')
              : require('~/assets/images/icon/expan-more-info_2.svg')
          "
          style="cursor: pointer;"
        />
      </v-hover>
    </span>
    <v-card
      v-if="!expandMeetingDetail && panel"
      flat
      class="mb-4 px-4 pt-4 pb-2"
    >
      <div class="d-flex justify-space-between mb-2">
        <div>
          <b class="title-info-object">Thông tin về cuộc họp</b>
          <v-icon
            v-if="
              ([
                statusTypes.APPROVED_STATUS,
                statusTypes.ENDED_STATUS,
                statusTypes.STREAMING_STATUS
              ].includes(data.status) &&
                $canPermission('manage-meeting.edit') &&
                $canPermission('manage-meeting.approve')) ||
                (data.status === statusTypes.NOT_APPROVED_STATUS &&
                  $canPermission('manage-meeting.edit'))
            "
            @click.stop="editMeeting()"
            class="ml-2 edit-pencil icon-hover"
            >mdi-lead-pencil</v-icon
          >
        </div>

        <v-hover v-slot:default="{ hover }">
          <img
            @click="panel = false"
            :src="
              hover
                ? require('~/assets/images/icon/expan-more-info_1__active.svg')
                : require('~/assets/images/icon/expan-more-info_1.svg')
            "
            style="cursor: pointer;"
          />
        </v-hover>
      </div>
      <div class="line-1-detail">
        <div class="mr-8 mb-2 d-inline-block">
          <img src="~assets/images/icon/clock.PNG" class="mr-1" />
          {{ infoObject.start_at | displayedTime }} -
          {{ infoObject.end_at | displayedTime }}
        </div>
        <div class="mr-8 mb-2 d-inline-block">
          <img src="~assets/images/icon/table.png" class="mr-1" />
          {{
            infoObject.meeting_room
              ? infoObject.meeting_room.name
              : 'Phòng họp không quản lý'
          }}
        </div>
        <div v-if="infoObject.main_member" class="mr-8 mb-2 d-inline-block">
          <img src="~assets/images/icon/hosted.png" class="mr-1" />
          Chủ trì cuộc họp:
          {{ infoObject.main_member.title + ' ' + infoObject.main_member.name }}
        </div>
        <div class="mb-2 d-inline-block">
          <img src="~assets/images/icon/approve_platform.png" class="mr-1" />
          {{ infoObject.status | getStatusMeeting }}
        </div>
      </div>
      <p class="mb-2">
        <span class="color-grey">Thành viên tham dự:</span>
        {{ infoObject.members.length }}
        <v-tooltip top max-width="300px">
          <template v-slot:activator="{ on, attrs }">
            <img
              v-bind="attrs"
              v-on="on"
              src="~assets/images/icon/info_ic.png"
              style="margin: 0 0 -5px 5px;"
            />
          </template>
          <p style="color: #000;">
            <b style="color: #4A4A4A;">Danh sách thành viên:</b>
            {{ infoObject.members | mapValueToItem }}
          </p>
        </v-tooltip>
      </p>
      <p>
        <span class="color-grey">Mô tả:</span>
        {{ infoObject.description }}
      </p>
    </v-card>
    <v-card style="background: #eef2f9;">
      <v-icon
        @click="toggleExpandMeetingDetail"
        class="ml-2 expand-meeting-detail icon-hover"
        >{{
          !expandMeetingDetail
            ? 'mdi-arrow-expand-all'
            : 'mdi-arrow-collapse-all'
        }}</v-icon
      >
      <v-tabs v-model="activeTab" class="v-tab__items">
        <v-tab key="audio" class="justify-space-between">
          Audio
          <span class="length-data">{{
            infoObject.hasOwnProperty('total_audio')
              ? infoObject.total_audio
              : 0
          }}</span>
          <v-btn
            v-if="
              activeTab === 0 &&
                canEditMeeting &&
                allowUploadFile &&
                !statusStream.isRecording &&
                $canPermission('manage-meeting.upload-file')
            "
            @click="showUploadAudioDialog"
            class="ml-4 blue"
            depressed
          >
            <div class="wrap-icon">
              <v-icon>mdi-upload</v-icon>
            </div>
            <span
              class="ml-1"
              style="text-transform: none; letter-spacing: normal"
            >
              Tải audio có sẵn
            </span>
          </v-btn>
        </v-tab>
        <v-tab-item key="audio">
          <vue-perfect-scrollbar
            ref="audioContent"
            class="message-wrap pl-3 pr-3"
          >
            <v-row>
              <v-col
                v-show="!expandAudioPlay && !expandMeetingDetail"
                md="4"
                sm="6"
                class="border-left-solid sm-half"
              >
                <card-list-audios
                  ref="listAudios"
                  :meeting-id="data.id"
                  :is-streaming="selectedAudio === -1"
                />
              </v-col>
              <v-col
                :class="expandMeetingDetail ? 'expand-audio-play' : ''"
                cols
                class="mt-5 pt-8 wrap-audio-play"
              >
                <v-btn
                  v-if="!expandMeetingDetail"
                  @click="startRecording()"
                  :class="selectedAudio === -1 ? 'active' : ''"
                  class="micro"
                >
                  <v-icon>mdi-microphone</v-icon>
                  {{ selectedAudio === -1 ? 'ĐANG GHI ÂM' : 'BẮT ĐẦU GHI ÂM' }}
                </v-btn>
                <v-card min-height="116" class="bg-audio-play">
                  <div class="pa-2">
                    <template v-if="selectedAudio === -1">
                      <record-audio
                        ref="recordAudioP"
                        :meeting-id="data.id"
                        :name="newRecordName"
                        @transcript="setAudioTrans"
                        @stop="stopRecording"
                      ></record-audio>
                    </template>
                    <template v-else>
                      <div
                        v-if="isEmpty(audioData)"
                        class="content-center mt-9"
                      >
                        <img
                          src="~assets/images/icon/radio_waves.svg"
                          class="mr-4"
                        />
                        Hiện tại không có file ghi âm
                      </div>
                      <div
                        v-else-if="
                          statusStream.isRecording &&
                            statusStream.audioFileId === selectedAudio
                        "
                        class="content-center mt-9"
                      >
                        <img
                          src="~assets/images/icon/radio_waves.svg"
                          class="mr-4"
                        />
                        Đang ghi âm trực tiếp.
                      </div>
                      <template v-else class="content-center">
                        <card-wave-suffer
                          ref="audioWave"
                          :current-time.sync="currentTime"
                        >
                        </card-wave-suffer>
                      </template>
                    </template>
                  </div>
                </v-card>
                <!--<v-icon-->
                <!--v-if="!isEmpty(audioData)"-->
                <!--@click="expandAudioPlay = !expandAudioPlay"-->
                <!--:class="expandAudioPlay ? 'blue' : ''"-->
                <!--class="ml-2 expand-audio-play-icon"-->
                <!--&gt;mdi-arrow-expand-all</v-icon-->
                <!--&gt;-->
              </v-col>
            </v-row>
          </vue-perfect-scrollbar>
        </v-tab-item>
      </v-tabs>
    </v-card>
    <div
      v-if="
        (!statusStream.isRecording ||
          statusStream.audioFileId !== selectedAudio) &&
          (selectedAudio === -1 || !isEmpty(audioData))
      "
    >
      <detail-audio-meeting
        :recordTranscript="recordTranscript"
        :audioFileId="audioFileId"
      />
    </div>
    <dialog-upload-audio
      ref="dialogUploadAudio"
      :meeting-id="data.id"
    ></dialog-upload-audio>
    <edit-dialog-meeting
      ref="editDialogMeeting"
      :edit-item="data"
      @submit-success="updateMeetingInfo"
    ></edit-dialog-meeting>
  </div>
</template>
<script>
import dayjs from 'dayjs'

import { isEmpty } from 'lodash'
import { mapGetters, mapMutations, mapState } from 'vuex'
import {
  APPROVED_STATUS,
  CANCELED_STATUS,
  DATE_FORMAT_SCHEDULE,
  ENDED_STATUS,
  NOT_APPROVED_STATUS,
  REJECT_STATUS
} from '../../../helpers/constant'
import { statusTypes } from '../../../constants/meeting'
import { DATE_FORMAT_MOMENT } from '~/helpers/constant'
import { absoluteUrl } from '~/helpers/common'
import MyBreadcrumbs from '~/components/common/MyBreadcrumbs'
import DetailAudioMeeting from '~/components/dialog/detailAudio/DetailAudioMeeting'
import CardListAudios from '~/components/card/CardListAudios'
import DialogUploadAudio from '~/components/dialog/DialogUploadAudio'
import EditDialogMeeting from '~/components/dialog/meetings/EditDialogMeeting'

export default {
  middleware: 'auth',
  components: {
    EditDialogMeeting,
    CardListAudios,
    MyBreadcrumbs,
    DetailAudioMeeting,
    DialogUploadAudio
  },
  head() {
    return {
      title: 'Thông tin chi tiết đối tượng'
    }
  },
  data() {
    return {
      panel: 0,
      activeTab: 0,
      currentTime: 0,
      expandAudioPlay: false,
      isPlayingSampleAudio: false,
      apiUrl: process.env.API_URL,

      newRecordName: '',
      recordTranscript: [],
      statusStream: {
        isRecording: false,
        audioFileId: -2
      },
      statusTypes,
      audioFileId: -1,
      intervalGetIdAudio: null
    }
  },
  computed: {
    ...mapState('layout', {
      expandMeetingDetail: 'expandMeetingDetail'
    }),
    ...mapState('meeting', {
      infoObject: 'infoObject',
      listAudio: 'audios',
      selectedAudio: 'selectedAudio'
    }),
    ...mapGetters({
      audioData: 'meeting/audioData'
    }),
    allowUploadFile() {
      return (
        this.infoObject.status === statusTypes.APPROVED_STATUS ||
        this.infoObject.status === statusTypes.ENDED_STATUS
      )
    },
    avatarUrl() {
      return absoluteUrl(this.data.avatar)
    },
    sampleAudioData() {
      const token = this.$auth.getToken('local').replace(/^(Bearer )/, '')
      return `${this.apiUrl}/meeting/get-audio/${this.data.id}?token=${token}`
    },
    canEditMeeting() {
      return this.infoObject.can_edit_meeting
    }
  },
  watch: {
    expandMeetingDetail(value) {
      try {
        this.$refs.audioWave.loadData()
      } catch (e) {}
    }
  },
  async asyncData({ params, store, error, $axios }) {
    try {
      const { data } = await $axios.post(`meeting/view/${params.id}`)
      store.commit('meeting/setInfoObject', data)
      return { data }
    } catch (e) {
      const code = e.response.status
      let msg = ''
      switch (code) {
        case 404:
          msg = 'Không tìm thấy cuộc họp này!'
          break
        case 403:
          msg = 'Bạn không có quyền xem cuộc họp này!'
          break
      }
      error({ statusCode: e.response.status, msg })
    }
  },
  filters: {
    displayedTime(timestamp) {
      return dayjs(timestamp).format('DD/MM/YYYY HH:mm')
    },
    mapValueToItem(value) {
      return value
        .map(function(elem) {
          return elem.name
        })
        .join(', ')
    },
    getStatusMeeting(status) {
      let stt = 'Chưa phê duyệt'
      switch (status) {
        case statusTypes.CANCELED_STATUS:
          stt = 'Đã hủy'
          break
        case statusTypes.APPROVED_STATUS:
          stt = 'Chưa họp'
          break

        case statusTypes.STREAMING_STATUS:
          stt = 'Đang họp'
          break

        case statusTypes.ENDED_STATUS:
          stt = 'Hoàn thành'
          break
      }
      return stt
    }
  },
  beforeRouteLeave(to, from, next) {
    // If the form is dirty and the user did not confirm leave,
    // prevent losing unsaved changes by canceling navigation
    if (
      this.statusStream.isRecording &&
      this.selectedAudio === -1 &&
      this.canEditMeeting
    ) {
      if (!this.confirmLeave()) {
        next(false)
      } else {
        // Navigate to next view
        this.$refs.recordAudioP.exitStream()
        next()
      }
    } else {
      next()
    }
  },
  mounted() {
    this.setSelectedAudio(-2)
    this.getStatusRoom()
    const vm = this
    this.newRecordName = `File ghi âm ${dayjs().format(DATE_FORMAT_MOMENT)}`

    // F2 => play, stop audio
    document.addEventListener('keydown', function(e) {
      if (e.keyCode === 113) {
        vm.$refs.audioWave.playPauseAudio()
      }
    })
  },
  methods: {
    ...mapMutations({
      toggleExpandMeetingDetail: 'layout/toggleExpandMeetingDetail',
      updateInfoObject: 'meeting/updateInfoObject',
      setSelectedAudio: 'meeting/setSelectedAudio',
      setInfoObject: 'meeting/setInfoObject'
    }),
    confirmLeave() {
      return window.confirm(
        'Cuộc họp đang ghi âm, bạn có  muốn kết thúc cuộc họp và chuyển sang chức năng khác?'
      )
    },
    async getStatusRoom() {
      try {
        const { data } = await this.$axios.post('device/get-status-room', {
          meetingId: this.infoObject.id,
          roomId: this.infoObject.room_id,
          type: this.infoObject.type
        })
        this.statusStream.isRecording = data.isRecording
        this.statusStream.audioFileId =
          data.audioFileId === -1 ? -2 : data.audioFileId
      } catch (e) {}
    },
    async stopRecording(reload) {
      await this.$axios.post('device/stop-recording', {
        type: 'recording',
        meetingId: this.infoObject.id,
        roomId: this.infoObject.room_id
      })
      this.statusStream.isRecording = false
      if (reload) {
        this.$refs.listAudios.reloadList()
      }
      this.updateInfoObject({ status: statusTypes.ENDED_STATUS })
      this.data.status = statusTypes.ENDED_STATUS
      window.addEventListener('beforeunload', function(e) {})
    },
    setAudioTrans(data) {
      // console.log(data)
      this.recordTranscript = data
      this.$nextTick(() => {
        if (this.$refs.messageWrap) {
          this.$refs.messageWrap.$el.scrollTop = 999999999
        }
      })
    },
    async startRecording() {
      if (this.cantRecordMeeting()) {
        return false
      }
      let fl = true
      let msg = 'Trình duyệt không cho phép ghi âm.'
      await navigator.mediaDevices
        .getUserMedia({ audio: true })
        .then(function(stream) {})
        .catch(function(err) {
          // console.log(err)
          fl = false
          if (err.name === 'NotFoundError') {
            msg = 'Vui lòng kết nối microphone để ghi âm.'
          }
        })
      if (!fl) {
        alert(msg)
      } else {
        this.sendRequestStartRecording()
      }
    },
    cantRecordMeeting() {
      const now = dayjs().format(DATE_FORMAT_SCHEDULE)
      if (this.statusStream.isRecording) {
        this.$snackbar.open({
          text: 'Cuộc họp đang ghi âm.',
          color: 'error'
        })
        return true
      }
      if (
        this.infoObject.status !== APPROVED_STATUS &&
        this.infoObject.status !== ENDED_STATUS
      ) {
        this.$snackbar.open({
          text: this.getMsgStatus(),
          color: 'error'
        })
        return true
      }
      if (this.infoObject.start_at > now) {
        this.$snackbar.open({
          text: 'Thời gian đặt lịch chưa đến.',
          color: 'error'
        })
        return true
      }
      if (this.infoObject.end_at <= now) {
        this.$snackbar.open({
          text: 'Thời gian đặt lịch đã hết hạn.',
          color: 'error'
        })
        return true
      }
      return false
    },
    async sendRequestStartRecording() {
      await this.$axios.post('device/change-status-recording', {
        type: 'recording',
        meetingId: this.infoObject.id,
        roomId: this.infoObject.room_id
      })
      this.statusStream.isRecording = true
      this.newRecordName = `File ghi âm ${dayjs().format(DATE_FORMAT_MOMENT)}`
      this.setSelectedAudio(-1)
      this.updateInfoObject({ status: statusTypes.STREAMING_STATUS })
      this.data.status = statusTypes.STREAMING_STATUS

      window.addEventListener('beforeunload', function(e) {
        e.preventDefault()
        e.returnValue = '123'
      })

      this.getFileId()
    },
    async getFileId() {
      try {
        const { data } = await this.$axios.post('device/get-status-room', {
          meetingId: this.infoObject.id,
          roomId: this.infoObject.room_id,
          type: this.infoObject.type
        })
        if (data.audioFileId >= 0) {
          this.audioFileId = data.audioFileId
          clearInterval(this.intervalGetIdAudio)
        } else {
          this.getFileId()
        }
      } catch (e) {
        this.getFileId()
      }
    },
    getMsgStatus() {
      let msg = ''
      switch (this.infoObject.status) {
        case REJECT_STATUS:
          msg = 'Cuộc họp đã bị từ chối.'
          break
        case CANCELED_STATUS:
          msg = 'Cuộc họp đã bị hủy.'
          break
        case NOT_APPROVED_STATUS:
          msg = 'Cuộc họp đã chưa được phê duyệt.'
          break
        // case ENDED_STATUS:
        //   msg = 'Cuộc họp đã kết thúc.'
        //   break
      }
      return msg
    },
    exportDetail(type) {
      this.$refs.detailVideo.exportDetail(type)
    },
    isEmpty(obj) {
      return isEmpty(obj)
    },
    playSampleAudio() {
      if (!this.data.template) {
        this.$snackbar.open({
          text: 'Không có file audio!',
          color: 'error'
        })
      }
      if (this.$refs.audio.currentTime > 0 && !this.$refs.audio.paused) {
        this.$refs.audio.pause()
        this.$refs.audio.currentTime = 0
        this.isPlayingSampleAudio = false
      } else {
        this.$refs.audio.play()
        this.isPlayingSampleAudio = true
      }
    },
    showUploadAudioDialog() {
      this.$refs.dialogUploadAudio.showUploadDialog()
    },
    showUploadVideoDialog() {
      this.$refs.dialogUploadVideo.showUploadDialog()
    },
    editMeeting() {
      this.$refs.editDialogMeeting.showDialog()
    },
    updateMeetingInfo(val) {
      this.data = val
      this.setInfoObject({ ...val })
    }
  }
}
</script>
