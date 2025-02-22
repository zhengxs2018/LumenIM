<script setup lang="ts">
import {
  reactive,
  watch,
  computed,
  nextTick,
  onMounted,
  inject,
  ref,
} from 'vue'
import { NDropdown, NCheckbox, NImageGroup } from 'naive-ui'
import { Loading, MoreThree, ToTop } from '@icon-park/vue-next'
import { publisher } from '@/utils/publisher.ts'
import socket from '@/socket'
import { useDialogueStore } from '@/store'
import { formatTime, parseTime } from '@/utils/datetime'
import { clipboard, htmlDecode, clipboardImage } from '@/utils/common'
import { downloadImage } from '@/utils/functions'
import { addClass, removeClass } from '@/utils/dom'
import { formatTalkRecord } from '@/utils/talk'
import { MessageComponents, ForwardableMessageType } from '@/constant/message'
import { ServeTalkRecords } from '@/api/chat'
import { useMenu } from './menu'
import SkipBottom from './SkipBottom.vue'
import { IMessageRecord } from '@/types/chat.ts'

const { dropdown, showDropdownMenu, closeDropdownMenu } = useMenu()
const user: any = inject('$user')
const dialogueStore = useDialogueStore()

const props = defineProps({
  uid: {
    type: Number,
    default: 0,
  },
  talk_type: {
    type: Number,
    default: 0,
  },
  receiver_id: {
    type: Number,
    default: 0,
  },
  index_name: {
    type: String,
    default: '',
  },
})

let locationMessage: any = null

// 对话记录
const records = computed((): IMessageRecord[] => dialogueStore.records)

// 加载配置
const loadConfig = reactive({
  status: 0,
  minRecord: 0,
})

// 置底按钮
const skipBottom = ref(false)

// 加载会话记录
const onLoadTalk = () => {
  const data = {
    record_id: loadConfig.minRecord,
    receiver_id: props.receiver_id,
    talk_type: props.talk_type,
    limit: 30,
  }

  if (locationMessage) {
    data.limit = 100
  }

  let scrollHeight = 0
  let el = document.getElementById('imChatPanel')
  if (el) {
    scrollHeight = el.scrollHeight
  }

  loadConfig.status = 0

  const response = ServeTalkRecords(data)
  response.then(res => {
    // 防止对话切换过快，数据渲染错误
    if (
      data.talk_type != props.talk_type ||
      data.receiver_id != props.receiver_id
    ) {
      locationMessage = null
      return
    }

    const records = res.data.items || []

    records.map((item: IMessageRecord) => formatTalkRecord(props.uid, item))

    // 判断是否是初次加载
    if (data.record_id == 0) {
      dialogueStore.clearDialogueRecord()
    }

    if (props.talk_type == 1) {
      onAfterRead(records)
    }

    dialogueStore.unshiftDialogueRecord(records.reverse())

    loadConfig.status = records.length >= res.data.limit ? 1 : 2
    loadConfig.minRecord = res.data.record_id

    nextTick(() => {
      if (!el) return

      if (data.record_id == 0) {
        el.scrollTop = el.scrollHeight
      } else {
        el.scrollTop = el.scrollHeight - scrollHeight
      }

      if (locationMessage) {
        onJumpMessage(locationMessage.msgid)
      }
    })
  })

  response.catch(() => {
    loadConfig.status = 1
  })
}

function onAfterRead(records: IMessageRecord[]) {
  let ids: number[] = []

  for (const record of records) {
    if (props.receiver_id === record.user_id && record.is_read === 0) {
      ids.push(record.id)
    }
  }

  if (ids.length) {
    socket.emit('im.message.read', {
      receiver_id: props.receiver_id,
      msg_id: ids,
    })
  }
}

// 是否显示消息时间
const isShowTalkTime = (index: number, datetime: string) => {
  if (datetime == undefined) {
    return false
  }

  if (records.value[index].is_revoke == 1) {
    return false
  }

  datetime = datetime.replace(/-/g, '/')
  let time = Math.floor(Date.parse(datetime) / 1000)
  let currTime = Math.floor(new Date().getTime() / 1000)

  // 当前时间5分钟内时间不显示
  if (currTime - time < 300) return false

  // 判断是否是最后一条消息,最后一条消息默认显示时间
  if (index == records.value.length - 1) {
    return true
  }

  let nextDate = records.value[index + 1].created_at.replace(/-/g, '/')

  return !(
    parseTime(new Date(datetime), '{y}-{m}-{d} {h}:{i}') ==
    parseTime(new Date(nextDate), '{y}-{m}-{d} {h}:{i}')
  )
}

// 窗口滚动事件
const onPanelScroll = (e: any) => {
  if (e.target.scrollTop == 0 && loadConfig.status == 1) {
    onLoadTalk()
  }

  const height = e.target.scrollTop + e.target.clientHeight

  skipBottom.value = height < e.target.scrollHeight - 200
  if (!skipBottom.value && dialogueStore.unreadBubble) {
    dialogueStore.setUnreadBubble(0)
  }

  // 检测是否到达底部
  if (skipBottom.value == false) {
    let len = dialogueStore.records.length

    if (len > 100) {
      // 为了优化数据过多页面卡顿，页面最多只显示100条数据
      // 目前没有用虚拟列表只能这么干
      dialogueStore.records.splice(0, len - 100)

      let minid = 0
      dialogueStore.records.forEach((item: IMessageRecord) => {
        if (minid == 0) {
          minid = item.sequence
        } else {
          if (item.sequence < minid) {
            minid = item.sequence
          }
        }
      })

      loadConfig.minRecord = minid
      loadConfig.status = 1
    }
  }
}

// 复制文本信息
const onCopyText = (data: IMessageRecord) => {
  if (data.content && data.content.length > 0) {
    return clipboard(htmlDecode(data.content), () =>
      window['$message'].success('复制成功')
    )
  }

  if (data.extra?.url) {
    return clipboardImage(data.extra.url, () => {
      window['$message'].success('复制成功')
    })
  }
}

// 删除对话消息
const onDeleteTalk = (data: IMessageRecord) => {
  dialogueStore.ApiDeleteRecord([data.id])
}

// 撤销对话消息
const onRevokeTalk = (data: IMessageRecord) => {
  dialogueStore.ApiRevokeRecord(data.id)
}

// 多选事件
const onMultiSelect = (data: IMessageRecord) => {
  dialogueStore.updateDialogueRecord({
    id: data.id,
    isCheck: true,
  })

  dialogueStore.isOpenMultiSelect = true
}

const onDownloadFile = (data: IMessageRecord) => {
  if (data.msg_type == 3) {
    return downloadImage(data.extra.url, `${data.msg_id}.${data.extra.suffix}`)
  }

  if (data.msg_type == 4) {
    return window['$message'].info('音频暂不支持下载!')
  }

  return window['$message'].info('视频暂不支持下载!')
}

const onQuoteMessage = (data: IMessageRecord) => {
  let item = {
    id: data.msg_id,
    title: `${data.nickname} ${data.created_at}`,
    describe: '',
    image: '',
  }

  switch (data.msg_type) {
    case 1:
      item.describe = data.content
      break // 文本消息
    case 2:
      item.describe = '[代码消息]'
      break // 代码消息
    case 3:
      item.image = data.extra.url
      break // 图片文件
    case 4:
      item.describe = '[语音文件]'
      break // 语音文件
    case 5:
      item.describe = '[视频文件]'
      break // 视频文件
    case 6:
      item.describe = '[其它文件]'
      break // 其它文件
    case 7:
      item.describe = '[位置消息]'
      break // 位置消息
    case 8:
      item.describe = '[名片消息]'
      break // 名片消息
    case 9:
      item.describe = '[转发消息]'
      break // 转发消息
    case 10:
      item.describe = '[登录消息]'
      break // 登录消息
    case 11:
      item.describe = '[投票消息]'
      break // 投票消息
    case 12:
      item.describe = '[图文消息]'
      break // 图文消息
  }

  publisher.emit('editor:quote', item)
}

const onClickNickname = (data: IMessageRecord) => {
  publisher.emit('editor:mention', {
    id: data.user_id,
    value: data.nickname,
  })
}

// 会话列表右键显示菜单
const onContextMenu = (e: any, item: IMessageRecord) => {
  if (!dialogueStore.isShowEditor || dialogueStore.isOpenMultiSelect) {
    return e.preventDefault()
  }

  showDropdownMenu(e, props.uid, item)

  e.preventDefault()
}

// 会话列表右键菜单回调事件
const onContextMenuHandle = (key: string) => {
  const evnets = {
    copy: onCopyText,
    revoke: onRevokeTalk,
    delete: onDeleteTalk,
    multiSelect: onMultiSelect,
    download: onDownloadFile,
    quote: onQuoteMessage,
  }

  // 触发事件
  evnets[key] && evnets[key](dropdown.item)

  closeDropdownMenu()
}

// 聊天版本滚动到底部
const onSkipBottom = () => {
  let el = document.getElementById('imChatPanel')
  if (el) {
    el.scrollTo({
      top: el.scrollHeight + 1000,
      behavior: 'smooth',
    })
  }
}

const onJumpMessage = (msgid: string) => {
  let element = document.getElementById(msgid)
  if (!element) {
    if (locationMessage === null) {
      locationMessage = {
        msgid: msgid,
        num: 3,
      }
    } else {
      locationMessage.num--

      if (locationMessage.num === 0) {
        locationMessage = null

        window['$message'].info('仅支持查看最近300条的记录')
        return
      }
    }

    let el = document.getElementById('imChatPanel')

    el?.scrollTo({
      top: 0,
      behavior: 'smooth',
    })
    return
  }

  locationMessage = null

  element?.scrollIntoView({
    behavior: 'smooth',
  })

  addClass(element, 'border')

  setTimeout(() => {
    element && removeClass(element, 'border')
  }, 3000)
}

const onReload = () => {
  loadConfig.status = 0
  loadConfig.minRecord = 0
  locationMessage = null

  onLoadTalk()
}

const onRowClick = (item: IMessageRecord) => {
  if (dialogueStore.isOpenMultiSelect) {
    if (ForwardableMessageType.includes(item.msg_type)) {
      item.isCheck = !item.isCheck
    } else {
      window['$message'].info('此类消息不支持转发')
    }
  }
}

watch(props, onReload)

onMounted(onReload)
</script>

<template>
  <section class="section">
    <NImageGroup>
      <div
        id="imChatPanel"
        class="me-scrollbar me-scrollbar-thumb talk-container"
        @scroll="onPanelScroll($event)"
      >
        <!-- 数据加载状态栏 -->
        <div class="load-toolbar pointer">
          <span v-if="loadConfig.status == 0"> 正在加载数据中 ... </span>
          <span v-else-if="loadConfig.status == 1" @click="onLoadTalk">
            查看更多消息 ...
          </span>
          <span v-else class="no-more"> 没有更多消息了 </span>
        </div>

        <div
          class="message-item"
          v-for="(item, index) in records"
          :key="item.msg_id"
          :id="item.msg_id"
        >
          <!-- 系统消息 -->
          <div v-if="item.msg_type >= 1000" class="message-box">
            <component
              :is="MessageComponents[item.msg_type] || 'unknown-message'"
              :extra="item.extra"
              :data="item"
            />
          </div>

          <!-- 撤回消息 -->
          <div v-else-if="item.is_revoke == 1" class="message-box">
            <revoke-message
              :login_uid="uid"
              :user_id="item.user_id"
              :nickname="item.nickname"
              :talk_type="item.talk_type"
              :datetime="item.created_at"
            />
          </div>

          <div
            v-else
            class="message-box record-box"
            :class="{
              'direction-rt': item.float == 'right',
              'multi-select': dialogueStore.isOpenMultiSelect,
              'multi-select-check': item.isCheck,
            }"
          >
            <!-- 多选按钮 -->
            <aside
              v-if="dialogueStore.isOpenMultiSelect"
              class="checkbox-column"
            >
              <n-checkbox
                size="small"
                :checked="item.isCheck"
                @update:checked="item.isCheck = !item.isCheck"
              />
            </aside>

            <!-- 头像信息 -->
            <aside class="avatar-column">
              <im-avatar
                class="pointer"
                :src="item.avatar"
                :size="30"
                :username="item.nickname"
                @click="user(item.user_id)"
              />
            </aside>

            <!-- 主体信息 -->
            <main class="main-column">
              <div class="talk-title">
                <span
                  class="nickname pointer"
                  v-show="talk_type == 2 && item.float == 'left'"
                  @click="onClickNickname(item)"
                >
                  {{ item.nickname }}
                </span>
                <span>{{ parseTime(item.created_at, '{m}/{d} {h}:{i}') }}</span>
              </div>

              <div
                class="talk-content"
                :class="{ pointer: dialogueStore.isOpenMultiSelect }"
                @click="onRowClick(item)"
              >
                <component
                  :is="MessageComponents[item.msg_type] || 'unknown-message'"
                  :extra="item.extra"
                  :data="item"
                  :max-width="true"
                  @contextmenu.prevent="onContextMenu($event, item)"
                />

                <div class="talk-tools">
                  <template v-if="talk_type == 1 && item.float == 'right'">
                    <loading
                      theme="outline"
                      size="19"
                      fill="#000"
                      :strokeWidth="1"
                      class="icon-rotate"
                      v-show="item.send_status == 1"
                    />

                    <span v-show="item.send_status == 1"> 正在发送... </span>
                    <!-- <span v-show="item.send_status != 1"> 已送达 </span> -->
                  </template>

                  <n-icon
                    class="more-tools pointer"
                    :component="MoreThree"
                    @click="onContextMenu($event, item)"
                  />
                </div>
              </div>

              <div
                v-if="item.extra.reply"
                class="talk-reply pointer"
                @click="onJumpMessage(item.extra?.reply?.msg_id)"
              >
                <n-icon :component="ToTop" size="14" class="icon-top" />
                <span class="ellipsis">
                  回复 {{ item.extra?.reply?.nickname }}:
                  {{ item.extra?.reply?.content }}
                </span>
              </div>
            </main>
          </div>

          <div class="datetime" v-show="isShowTalkTime(index, item.created_at)">
            {{ formatTime(item.created_at) }}
          </div>
        </div>
      </div>
    </NImageGroup>

    <!-- 置底按钮 -->
    <SkipBottom v-model="skipBottom" @click="onSkipBottom" />
  </section>

  <!-- 右键菜单 -->
  <n-dropdown
    :show="dropdown.show"
    :x="dropdown.x"
    :y="dropdown.y"
    :options="dropdown.options"
    @select="onContextMenuHandle"
    @clickoutside="closeDropdownMenu"
  />
</template>

<style lang="less" scoped>
.icon-rotate {
  animation: rotate 1s linear infinite;
}

@keyframes rotate {
  to {
    transform: rotate(360deg);
  }
}

.section {
  position: relative;
  height: 100%;
  width: 100%;
  overflow: hidden;
}

.talk-container {
  height: 100%;
  width: 100%;
  box-sizing: border-box;
  padding: 10px 15px 30px;
  overflow-y: auto;
  overflow-x: hidden;

  .load-toolbar {
    height: 38px;
    color: #409eff;
    text-align: center;
    line-height: 38px;
    font-size: 13px;
    .no-more {
      color: #b9b3b3;
    }
  }

  .message-item {
    &.border {
      border-radius: 10px;
      border: 1px solid var(--im-primary-color);
    }
  }

  .message-box {
    width: 100%;
    min-height: 30px;
    margin-bottom: 5px;
  }

  .datetime {
    height: 30px;
    line-height: 30px;
    color: #ccc9c9;
    font-size: 12px;
    text-align: center;
    margin: 5px 0;
  }

  .record-box {
    display: flex;
    flex-direction: row;

    .checkbox-column {
      display: flex;
      justify-content: center;
      width: 35px;
      order: 1;
      user-select: none;
      padding-top: 12px;
    }

    .avatar-column {
      width: 35px;
      display: flex;
      align-items: center;
      order: 2;
      user-select: none;
      padding-top: 10px;
      flex-direction: column;
    }

    .main-column {
      flex: 1 auto;
      order: 3;
      position: relative;
      box-sizing: border-box;
      padding: 5px;
      overflow: hidden;
      min-height: 30px;

      .talk-title {
        display: flex;
        align-items: center;
        height: 24px;
        margin-bottom: 2px;
        font-size: 12px;
        user-select: none;
        color: #a7a0a0;
        transition: 0.5s ease;
        opacity: 1;

        &.show {
          opacity: 1;
        }

        .nickname {
          color: var(--im-text-color);
          margin-right: 5px;
        }

        span {
          transform: scale(0.88);
          transform-origin: left center;
        }
      }

      .talk-content {
        display: flex;
        justify-content: flex-start;
        align-items: flex-end;

        box-sizing: border-box;
        width: 100%;

        .talk-tools {
          display: flex;
          margin: 0 8px;
          color: #a79e9e;
          font-size: 12px;
          user-select: none;
          align-items: center;
          justify-content: space-around;

          .more-tools {
            display: none;
            margin-left: 5px;
          }
        }
      }

      .talk-reply {
        display: flex;
        align-items: flex-start;
        align-items: center;
        width: fit-content;
        padding: 4px;
        margin-top: 3px;
        margin-right: auto;
        font-size: 12px;
        color: #8f8f8f;
        word-break: break-all;
        background-color: var(--im-message-left-bg-color);
        border-radius: 5px;
        max-width: 300px;
        overflow: hidden;
        user-select: none;

        .icon-top {
          margin-right: 3px;
        }

        .ellipsis {
          display: -webkit-inline-box;
          text-overflow: ellipsis;
          -webkit-line-clamp: 3;
          -webkit-box-orient: vertical;
        }
      }

      &:hover {
        .talk-title {
          opacity: 1;
        }

        .more-tools {
          display: flex !important;
        }
      }
    }

    &.direction-rt {
      .avatar-column {
        order: 3;
      }

      .main-column {
        order: 2;

        .talk-title {
          justify-content: flex-end;

          span {
            transform-origin: right center;
          }
        }

        .talk-content {
          flex-direction: row-reverse;
        }

        .talk-reply {
          margin-right: 0;
          margin-left: auto;
        }
      }
    }

    &.multi-select {
      border-radius: 5px;

      &:hover,
      &.multi-select-check {
        background-color: var(--im-active-bg-color);
      }
    }
  }
}
</style>
