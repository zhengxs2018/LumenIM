<script setup lang="ts">
import { computed, ref } from 'vue'
import ForwardRecord from '../ForwardRecord.vue'
import { ForwardExtra } from './types'

const props = defineProps<{
  extra: ForwardExtra
  data: any
  pid: string
  maxWidth: Boolean
}>()

const isShowRecord = ref(false)

const title = computed(() => {
  return [...new Set(props.extra.records.map(v => v.nickname))].join('、')
})

const onClick = () => {
  isShowRecord.value = true
}

let pids = props.pid
if (pids == '' || pids == undefined) {
  pids = props.data.id
} else {
  pids = `${pids},${props.data.id}`
}
</script>
<template>
  <section class="im-message-forward pointer" @click="onClick">
    <div class="title">{{ title }} 的会话记录</div>
    <div class="list" v-for="(record, index) in extra.records" :key="index">
      <p>
        <span>{{ record.nickname }}: </span>
        <span>{{ record.text }}</span>
      </p>
    </div>

    <div class="tips">
      <span>转发：聊天会话记录 ({{ extra.msg_ids.length }}条)</span>
    </div>

    <ForwardRecord
      v-if="isShowRecord"
      :record-id="data.id"
      :pid="pids"
      @close="isShowRecord = false"
    />
  </section>
</template>

<style lang="less" scoped>
.im-message-forward {
  width: 250px;
  min-height: 95px;
  max-height: 150px;
  border-radius: 10px;
  padding: 8px 10px;
  border: 1px solid var(--im-message-border-color);
  user-select: none;

  .title {
    height: 30px;
    line-height: 30px;
    font-size: 15px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    font-weight: 400;
    margin-bottom: 5px;
  }

  .list p {
    height: 18px;
    line-height: 18px;
    font-size: 12px;
    color: #a8a8a8;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    margin-bottom: 5px;
  }

  .tips {
    height: 32px;
    line-height: 35px;
    color: #8a8888;
    border-top: 1px solid var(--border-color);
    font-size: 12px;
    margin-top: 12px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
}
</style>
