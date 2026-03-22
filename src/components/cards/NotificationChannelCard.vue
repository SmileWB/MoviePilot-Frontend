<script setup lang="ts">
import { ref, computed, watch, onMounted } from 'vue'
import { NotificationConf } from '@/api/types'
import { getLogoUrl } from '@/utils/imageUtils'
import { useToast } from 'vue-toastification'
import { cloneDeep } from 'lodash-es'
import { useI18n } from 'vue-i18n'
import { useDisplay } from 'vuetify'
import api from '@/api'

// 显示器宽度
const display = useDisplay()

const { t } = useI18n()

// 定义输入
const props = defineProps({
  // 单个通知
  notification: {
    type: Object as PropType<NotificationConf>,
    required: true,
  },
  // 所有通知
  notifications: {
    type: Array as PropType<NotificationConf[]>,
    required: true,
  },
})

// 定义触发的自定义事件
const emit = defineEmits(['close', 'change', 'done'])

// 提示框
const $toast = useToast()

// 通知详情弹窗
const notificationInfoDialog = ref(false)

// 通知详情
const notificationInfo = ref<NotificationConf>({
  name: '',
  type: '',
  enabled: false,
  config: {},
})

// 测试发送消息对话框
const testMessageDialog = ref(false)

// 测试消息内容
const testMessage = ref({
  title: '测试消息',
  content: '这是一条测试消息，如果您能收到此消息，说明通知渠道配置正确。',
})

// 正在发送测试消息
const sendingTestMessage = ref(false)

// 各通知类型的名称字典
const notificationTypeNames: { [key: string]: string } = {
  wechat: t('notification.wechat.name'),
  telegram: t('notification.telegram.name'),
  qqbot: t('notification.qqbot.name'),
  vocechat: t('notification.vocechat.name'),
  synologychat: t('notification.synologychat.name'),
  slack: t('notification.slack.name'),
  discord: t('notification.discord.name'),
  webpush: t('notification.webpush.name'),
  feishu: '飞书',
  custom: t('setting.notification.custom'),
}

// 消息类型下拉字典
const notificationTypes = [
  { value: '资源下载', title: t('notificationSwitch.resourceDownload') },
  { value: '整理入库', title: t('notificationSwitch.organize') },
  { value: '订阅', title: t('notificationSwitch.subscribe') },
  { value: '站点', title: t('notificationSwitch.site') },
  { value: '媒体服务器', title: t('notificationSwitch.mediaServer') },
  { value: '手动处理', title: t('notificationSwitch.manual') },
  { value: '插件', title: t('notificationSwitch.plugin') },
  { value: '其它', title: t('notificationSwitch.other') },
]

function ensureWechatConfigDefaults(notification: NotificationConf) {
  if (notification.type !== 'wechat') {
    return
  }
  if (!notification.config) {
    notification.config = {}
  }
  if (!notification.config.WECHAT_MODE) {
    notification.config.WECHAT_MODE = 'app'
  }
  if (!notification.config.WECHAT_BOT_WS_URL) {
    notification.config.WECHAT_BOT_WS_URL = 'wss://openws.work.weixin.qq.com'
  }
}

const isWechatBotMode = computed({
  get: () => notificationInfo.value.config?.WECHAT_MODE === 'bot',
  set: value => {
    if (!notificationInfo.value.config) {
      notificationInfo.value.config = {}
    }
    notificationInfo.value.config.WECHAT_MODE = value ? 'bot' : 'app'
    if (value && !notificationInfo.value.config.WECHAT_BOT_WS_URL) {
      notificationInfo.value.config.WECHAT_BOT_WS_URL = 'wss://openws.work.weixin.qq.com'
    }
  },
})

function ensureFeishuConfigDefaults(notification: NotificationConf) {
  if (notification.type !== 'feishu') {
    return
  }
  if (!notification.config) {
    notification.config = {}
  }
  if (!notification.config.FEISHU_MODE) {
    notification.config.FEISHU_MODE = 'sdk'
  }
}

const feishuMode = computed({
  get: () => notificationInfo.value.config?.FEISHU_MODE || 'sdk',
  set: value => {
    if (!notificationInfo.value.config) {
      notificationInfo.value.config = {}
    }
    notificationInfo.value.config.FEISHU_MODE = value
  },
})

// 飞书长连接状态
const feishuWsStatus = ref({
  connected: false,
  text: '加载中...',
  color: 'info',
  icon: 'mdi-loading',
  showReconnect: false
})

// 重连中状态
const reconnecting = ref(false)

// 加载飞书长连接状态
async function loadFeishuWsStatus() {
  if (notificationInfo.value.type !== 'feishu' || notificationInfo.value.config?.FEISHU_MODE !== 'sdk') {
    return
  }

  try {
    // 查询实际连接状态
    const result: { [key: string]: any } = await api.get(
      `system/notification/ws-status?channel=${encodeURIComponent(notificationInfo.value.name)}`
    )

    if (result.success && result.data) {
      const data = result.data
      if (data.connected) {
        feishuWsStatus.value = {
          connected: true,
          text: `长连接已运行`,
          color: 'success',
          icon: 'mdi-check-circle',
          showReconnect: false
        }
      } else {
        feishuWsStatus.value = {
          connected: false,
          text: data.status === 'not_sdk_mode'
            ? 'Webhook 模式'
            : data.status === 'sdk_not_installed'
              ? 'SDK 未安装'
              : '未连接',
          color: data.status === 'not_sdk_mode' ? 'info' : 'warning',
          icon: data.status === 'not_sdk_mode' ? 'mdi-information' : 'mdi-alert-circle',
          showReconnect: data.status !== 'not_sdk_mode' && data.status !== 'sdk_not_installed'
        }
      }
    } else {
      feishuWsStatus.value = {
        connected: false,
        text: result.message || '查询失败',
        color: 'error',
        icon: 'mdi-alert-circle',
        showReconnect: true
      }
    }
  } catch (error) {
    console.error(error)
    feishuWsStatus.value = {
      connected: false,
      text: '查询失败',
      color: 'error',
      icon: 'mdi-alert-circle',
      showReconnect: true
    }
  }
}

// 处理飞书重连
async function handleFeishuReconnect() {
  if (notificationInfo.value.type !== 'feishu' || notificationInfo.value.config?.FEISHU_MODE !== 'sdk') {
    return
  }

  reconnecting.value = true
  try {
    const result: { [key: string]: any } = await api.post(
      `system/notification/ws-reconnect?channel=${encodeURIComponent(notificationInfo.value.name)}`,
      null
    )

    if (result.success) {
      $toast.success('重连成功，请稍后查看状态')
      // 延迟刷新状态
      setTimeout(() => loadFeishuWsStatus(), 3000)
    } else {
      $toast.error(result.message || '重连失败')
    }
  } catch (error) {
    console.error(error)
    $toast.error('重连失败：' + (error.message || '未知错误'))
  } finally {
    reconnecting.value = false
  }
}


// 监听飞书模式变化，重置状态
watch(feishuMode, () => {
  feishuWsStatus.value = {
    connected: false,
    text: '加载中...',
    color: 'info',
    icon: 'mdi-loading',
    showReconnect: false
  }
  loadFeishuWsStatus()
})

// 监听长连接 URL 变化，重新加载状态
watch(() => notificationInfo.value.config?.FEISHU_BOT_WS_URL, () => {
  if (notificationInfo.value.type === 'feishu' && notificationInfo.value.config?.FEISHU_MODE === 'sdk') {
    feishuWsStatus.value = {
      connected: false,
      text: '加载中...',
      color: 'info',
      icon: 'mdi-loading',
      showReconnect: false
    }
    loadFeishuWsStatus()
  }
})


// 打开详情弹窗
function openNotificationInfoDialog() {
  // 替换成深复制，避免修改时影响原数据
  notificationInfo.value = cloneDeep(props.notification)
  ensureWechatConfigDefaults(notificationInfo.value)
  ensureFeishuConfigDefaults(notificationInfo.value)
  notificationInfoDialog.value = true
  // 如果是飞书 SDK 模式，加载长连接状态
  if (notificationInfo.value.type === 'feishu' && notificationInfo.value.config?.FEISHU_MODE === 'sdk') {
    // 重置状态为加载中
    feishuWsStatus.value = {
      connected: false,
      text: '加载中...',
      color: 'info',
      icon: 'mdi-loading',
      showReconnect: false
    }
    // 等待下一帧渲染后再请求状态
    setTimeout(() => loadFeishuWsStatus(), 100)
  }
}

// 加载初始状态（用于卡片显示）
function loadInitialFeishuStatus() {
  if (props.notification.type === 'feishu' && props.notification.config?.FEISHU_MODE === 'sdk') {
    loadFeishuWsStatus()
  }
}

// 组件挂载时加载初始状态
onMounted(() => {
  loadInitialFeishuStatus()
})

// 保存详情数据
async function saveNotificationInfo() {
  // 为空不保存，跳出警告框
  if (!notificationInfo.value.name) {
    $toast.error(t('notification.name') + t('common.required'))
    return
  }
  // 重名判断
  if (props.notifications.some(item => item.name === notificationInfo.value.name && item !== props.notification)) {
    $toast.error(t('notification.channel') + `【${notificationInfo.value.name}】` + t('common.exists'))
    return
  }
  ensureWechatConfigDefaults(notificationInfo.value)
  ensureFeishuConfigDefaults(notificationInfo.value)

  // 如果是飞书 SDK 模式，保存后自动重连
  const shouldReconnect = notificationInfo.value.type === 'feishu'
    && notificationInfo.value.config?.FEISHU_MODE === 'sdk'

  notificationInfoDialog.value = false
  emit('change', notificationInfo.value, props.notification.name)
  emit('done')

  // 保存后重连
  if (shouldReconnect) {
    reconnecting.value = true
    try {
      const result: { [key: string]: any } = await api.post(
        `system/notification/ws-reconnect?channel=${encodeURIComponent(notificationInfo.value.name)}`,
        null
      )
      if (result.success) {
        $toast.success('配置已保存，长连接正在重新建立')
      } else {
        $toast.warning('配置已保存，但长连接重连失败：' + (result.message || '未知错误'))
      }
    } catch (error) {
      console.error(error)
      $toast.warning('配置已保存，但长连接重连失败')
    } finally {
      reconnecting.value = false
    }
  }
}

// 根据存储类型选择图标
const getIcon = computed(() => {
  switch (props.notification.type) {
    case 'wechat':
      return getLogoUrl('wechat')
    case 'telegram':
      return getLogoUrl('telegram')
    case 'qqbot':
      return getLogoUrl('qq')
    case 'vocechat':
      return getLogoUrl('vocechat')
    case 'synologychat':
      return getLogoUrl('synologychat')
    case 'slack':
      return getLogoUrl('slack')
    case 'discord':
      return getLogoUrl('discord')
    case 'webpush':
      return getLogoUrl('chrome')
    case 'feishu':
      return getLogoUrl('feishu')
    default:
      return getLogoUrl('notification')
  }
})

// 按钮点击
function onClose() {
  emit('close')
}

// 打开测试消息对话框
function openTestMessageDialog() {
  notificationInfo.value = cloneDeep(props.notification)
  testMessageDialog.value = true
}

// 发送测试消息
async function sendTestMessage() {
  if (!notificationInfo.value.name) {
    $toast.error(t('notification.channel') + t('common.required'))
    return
  }

  sendingTestMessage.value = true
  try {
    const result: { [key: string]: any } = await api.post('system/notification/test', {
      channel: notificationInfo.value.name,
      title: testMessage.value.title,
      content: testMessage.value.content,
    })

    if (result.success) {
      $toast.success(t('common.testSendSuccess'))
      testMessageDialog.value = false
    } else {
      $toast.error(result.message || t('common.testSendFailed'))
    }
  } catch (error) {
    console.error(error)
    $toast.error(t('common.testSendFailed'))
  } finally {
    sendingTestMessage.value = false
  }
}
</script>
<template>
  <div>
    <VCard variant="tonal" @click="openNotificationInfoDialog">
      <span class="absolute top-3 right-12">
        <IconBtn>
          <VIcon class="cursor-move" icon="mdi-drag" />
        </IconBtn>
      </span>
      <VDialogCloseBtn @click="onClose" />
      <VCardText class="flex justify-space-between align-center gap-3">
        <div class="align-self-start">
          <div class="flex items-center">
            <VBadge v-if="props.notification.enabled" dot inline color="success" class="me-1" />
            <span class="text-h6">{{ props.notification.name }}</span>
            <!-- 飞书 SDK 模式长连接状态 -->
            <VIcon
              v-if="notification.type === 'feishu' && notification.config?.FEISHU_MODE === 'sdk'"
              :icon="feishuWsStatus.icon"
              :color="feishuWsStatus.color"
              size="small"
              class="ms-2"
            />
          </div>
          <div class="text-body-1 mb-3">{{ notificationTypeNames[notification.type] }}</div>
        </div>
        <VImg :src="getIcon" cover class="mt-7 me-1" max-width="3rem" />
      </VCardText>
    </VCard>

    <VDialog
      v-if="notificationInfoDialog"
      v-model="notificationInfoDialog"
      scrollable
      max-width="40rem"
      :fullscreen="!display.mdAndUp.value"
    >
      <VCard>
        <VCardItem class="py-2">
          <template #prepend>
            <VIcon icon="mdi-cog" class="me-2" />
          </template>
          <VCardTitle>{{ t('common.config') }}</VCardTitle>
          <VCardSubtitle>{{ props.notification.name }}</VCardSubtitle>
        </VCardItem>
        <VDialogCloseBtn @click="notificationInfoDialog = false" />
        <VDivider />
        <VCardText>
          <VForm>
            <VRow>
              <VCol cols="12" md="6">
                <VSwitch v-model="notificationInfo.enabled" :label="t('notification.enabled')" />
              </VCol>
              <VCol cols="12">
                <VAutocomplete
                  v-model="notificationInfo.switchs"
                  :items="notificationTypes"
                  :label="t('notification.type')"
                  :hint="t('notification.typeHint')"
                  multiple
                  clearable
                  chips
                  persistent-hint
                  prepend-inner-icon="mdi-bell-outline"
                />
              </VCol>
            </VRow>
            <VRow v-if="notificationInfo.type == 'wechat'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VSwitch
                  v-model="isWechatBotMode"
                  :label="t('notification.wechat.useBotMode')"
                  :hint="t('notification.wechat.useBotModeHint')"
                  persistent-hint
                  color="primary"
                />
              </VCol>
              <template v-if="isWechatBotMode">
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_BOT_ID"
                    :label="t('notification.wechat.botId')"
                    :hint="t('notification.wechat.botIdHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-robot"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_BOT_SECRET"
                    :label="t('notification.wechat.botSecret')"
                    :hint="t('notification.wechat.botSecretHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-key"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_BOT_CHAT_ID"
                    :label="t('notification.wechat.botChatId')"
                    :placeholder="t('notification.wechat.botChatIdPlaceholder')"
                    :hint="t('notification.wechat.botChatIdHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-chat-processing"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_BOT_WS_URL"
                    :label="t('notification.wechat.botWsUrl')"
                    :hint="t('notification.wechat.botWsUrlHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-lan-connect"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_ADMINS"
                    :label="t('notification.wechat.admins')"
                    :placeholder="t('notification.wechat.adminsPlaceholder')"
                    :hint="t('notification.wechat.adminsHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-account-supervisor"
                  />
                </VCol>
              </template>
              <template v-else>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_CORPID"
                    :label="t('notification.wechat.corpId')"
                    :hint="t('notification.wechat.corpIdHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-domain"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_APP_ID"
                    :label="t('notification.wechat.appId')"
                    :hint="t('notification.wechat.appIdHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-application"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_APP_SECRET"
                    :label="t('notification.wechat.appSecret')"
                    :hint="t('notification.wechat.appSecretHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-key"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_PROXY"
                    :label="t('notification.wechat.proxy')"
                    :hint="t('notification.wechat.proxyHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-server-network"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_TOKEN"
                    :label="t('notification.wechat.token')"
                    :hint="t('notification.wechat.tokenHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-key-variant"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_ENCODING_AESKEY"
                    :label="t('notification.wechat.encodingAesKey')"
                    :hint="t('notification.wechat.encodingAesKeyHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-lock"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.WECHAT_ADMINS"
                    :label="t('notification.wechat.admins')"
                    :placeholder="t('notification.wechat.adminsPlaceholder')"
                    :hint="t('notification.wechat.adminsHint')"
                    persistent-hint
                    prepend-inner-icon="mdi-account-supervisor"
                  />
                </VCol>
              </template>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'telegram'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.TELEGRAM_TOKEN"
                  :label="t('notification.telegram.token')"
                  :hint="t('notification.telegram.tokenHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-key"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.TELEGRAM_CHAT_ID"
                  :label="t('notification.telegram.chatId')"
                  :hint="t('notification.telegram.chatIdHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-chat"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.TELEGRAM_USERS"
                  :label="t('notification.telegram.users')"
                  :placeholder="t('notification.telegram.usersPlaceholder')"
                  :hint="t('notification.telegram.usersHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-account-group"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.TELEGRAM_ADMINS"
                  :label="t('notification.telegram.admins')"
                  :placeholder="t('notification.telegram.adminsPlaceholder')"
                  :hint="t('notification.telegram.adminsHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-account-supervisor"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.API_URL"
                  :label="t('notification.telegram.apiUrl')"
                  :placeholder="t('notification.telegram.apiUrlPlaceholder')"
                  :hint="t('notification.telegram.apiUrlHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-web"
                />
              </VCol>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'slack'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.SLACK_OAUTH_TOKEN"
                  :label="t('notification.slack.oauthToken')"
                  :placeholder="t('notification.slack.oauthTokenPlaceholder')"
                  :hint="t('notification.slack.oauthTokenHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-key"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.SLACK_APP_TOKEN"
                  :label="t('notification.slack.appToken')"
                  :placeholder="t('notification.slack.appTokenPlaceholder')"
                  :hint="t('notification.slack.appTokenHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-application"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.SLACK_CHANNEL"
                  :label="t('notification.slack.channel')"
                  :placeholder="t('notification.slack.channelPlaceholder')"
                  :hint="t('notification.slack.channelHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-pound"
                />
              </VCol>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'discord'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.DISCORD_BOT_TOKEN"
                  :label="t('notification.discord.botToken')"
                  :hint="t('notification.discord.botTokenHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-key-variant"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.DISCORD_GUILD_ID"
                  :label="t('notification.discord.guildId')"
                  :placeholder="t('notification.discord.guildIdPlaceholder')"
                  :hint="t('notification.discord.guildIdHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-pound"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.DISCORD_CHANNEL_ID"
                  :label="t('notification.discord.channelId')"
                  :placeholder="t('notification.discord.channelIdPlaceholder')"
                  :hint="t('notification.discord.channelIdHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-pound-box"
                />
              </VCol>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'synologychat'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.SYNOLOGYCHAT_WEBHOOK"
                  :label="t('notification.synologychat.webhook')"
                  :hint="t('notification.synologychat.webhookHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-webhook"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.SYNOLOGYCHAT_TOKEN"
                  :label="t('notification.synologychat.token')"
                  :hint="t('notification.synologychat.tokenHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-key"
                />
              </VCol>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'vocechat'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.VOCECHAT_HOST"
                  :label="t('notification.vocechat.host')"
                  :hint="t('notification.vocechat.hostHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-server"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.VOCECHAT_API_KEY"
                  :label="t('notification.vocechat.apiKey')"
                  :hint="t('notification.vocechat.apiKeyHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-key"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.VOCECHAT_CHANNEL_ID"
                  :label="t('notification.vocechat.channelId')"
                  :placeholder="t('notification.vocechat.channelIdPlaceholder')"
                  :hint="t('notification.vocechat.channelIdHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-pound"
                />
              </VCol>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'qqbot'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.QQ_APP_ID"
                  :label="t('notification.qqbot.appId')"
                  :hint="t('notification.qqbot.appIdHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-application"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.QQ_APP_SECRET"
                  :label="t('notification.qqbot.appSecret')"
                  :hint="t('notification.qqbot.appSecretHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-key"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.QQ_OPENID"
                  :label="t('notification.qqbot.openId')"
                  :placeholder="t('notification.qqbot.openIdPlaceholder')"
                  :hint="t('notification.qqbot.openIdHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-account"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.QQ_GROUP_OPENID"
                  :label="t('notification.qqbot.groupOpenId')"
                  :placeholder="t('notification.qqbot.groupOpenIdPlaceholder')"
                  :hint="t('notification.qqbot.groupOpenIdHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-account-group"
                />
              </VCol>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'webpush'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.config.WEBPUSH_USERNAME"
                  :label="t('notification.webpush.username')"
                  :hint="t('notification.webpush.usernameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-account"
                />
              </VCol>
            </VRow>
            <VRow v-else-if="notificationInfo.type == 'feishu'">
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :placeholder="t('notification.name')"
                  :hint="t('notification.nameHint')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VSelect
                  v-model="feishuMode"
                  :items="[
                    { title: 'SDK 模式（推荐）', value: 'sdk' },
                    { title: 'Webhook 模式', value: 'webhook' }
                  ]"
                  :label="'配置模式'"
                  :hint="'SDK 模式支持双向交互，Webhook 模式仅支持发送'"
                  persistent-hint
                  prepend-inner-icon="mdi-cog"
                />
              </VCol>
              <template v-if="feishuMode === 'sdk'">
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.FEISHU_BOT_APP_ID"
                    :label="'App ID（必填）'"
                    :placeholder="'APP_ID'"
                    :hint="'飞书应用 App ID，从飞书开放平台获取'"
                    persistent-hint
                    prepend-inner-icon="mdi-application"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.FEISHU_BOT_APP_SECRET"
                    :label="'App Secret（必填）'"
                    :placeholder="'APP_SECRET'"
                    :hint="'飞书应用 App Secret，从飞书开放平台获取'"
                    persistent-hint
                    prepend-inner-icon="mdi-key-variant"
                  />
                </VCol>
                <VCol cols="12" md="12">
                  <VTextField
                    v-model="notificationInfo.config.FEISHU_DEFAULT_USER_ID"
                    :label="'默认通知用户 Open ID（必填）'"
                    :placeholder="'ou_xxxxx'"
                    :hint="'测试消息或无目标用户时的默认通知对象，可通过「绑定默认通知用户」命令自动绑定'"
                    persistent-hint
                    prepend-inner-icon="mdi-account"
                  />
                </VCol>
                <VCol cols="12" md="12">
                  <VTextField
                    v-model="notificationInfo.config.FEISHU_BOT_WS_URL"
                    :label="'飞书长连接 WebSocket URL（可选）'"
                    :placeholder="'wss://***.feishu.cn/***.ws?***'"
                    :hint="'在飞书开放平台事件订阅页面开启长连接后复制 WebSocket 地址，用于接收用户消息'"
                    persistent-hint
                    prepend-inner-icon="mdi-websocket"
                  />
                </VCol>
                <VCol cols="12" md="12">
                  <VAlert
                    type="info"
                    variant="tonal"
                    :title="'长连接状态'"
                    :text="feishuWsStatus.text"
                    :color="feishuWsStatus.color"
                  >
                    <template #prepend>
                      <VIcon :icon="feishuWsStatus.icon" />
                    </template>
                    <template v-if="feishuWsStatus.showReconnect" #append>
                      <VBtn
                        size="small"
                        variant="text"
                        @click="handleFeishuReconnect"
                        :loading="reconnecting"
                      >
                        重新连接
                      </VBtn>
                    </template>
                  </VAlert>
                </VCol>
                <VCol cols="12" md="12">
                  <VAlert
                    type="info"
                    variant="tonal"
                    title="SDK 模式说明"
                    text="1. 在飞书开放平台创建企业内部应用，获取 App ID 和 App Secret（必填）  2. 配置默认通知用户 Open ID（必填，可通过飞书 API 获取或发送「绑定默认通知用户」命令自动绑定）  3. 可选配置 WebSocket URL 接收用户消息（长连接模式，在飞书开放平台事件订阅页面开启后复制 URL）  4. 不配置 WebSocket URL 时，SDK 模式仍可正常发送通知消息  5. 已集成飞书 SDK 核心代码，无需额外安装依赖"
                  />
                </VCol>
              </template>
              <template v-else-if="feishuMode === 'webhook'">
                <VCol cols="12" md="12">
                  <VTextField
                    v-model="notificationInfo.config.FEISHU_BOT_WEBHOOK"
                    :label="'飞书机器人 Webhook URL（必填）'"
                    :placeholder="'https://open.feishu.cn/open-apis/bot/v2/hook/xxx'"
                    :hint="'飞书机器人 Webhook 地址，创建机器人后从群聊设置中复制'"
                    persistent-hint
                    prepend-inner-icon="mdi-webhook"
                  />
                </VCol>
                <VCol cols="12" md="6">
                  <VTextField
                    v-model="notificationInfo.config.FEISHU_BOT_SECRET"
                    :label="'签名密钥（可选）'"
                    :placeholder="'可选，用于消息加签验证'"
                    :hint="'飞书机器人配置的签名密钥，用于验证消息来源，建议开启'"
                    persistent-hint
                    prepend-inner-icon="mdi-key"
                  />
                </VCol>
                <VCol cols="12" md="12">
                  <VAlert
                    type="warning"
                    variant="tonal"
                    title="Webhook 模式说明"
                    text="1. Webhook 模式仅支持向群聊发送消息  2. 不支持接收用户消息和按钮回调  3. 需要在群聊中添加机器人  4. 适合简单的通知场景"
                  />
                </VCol>
              </template>
            </VRow>
            <VRow v-else>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.type"
                  :label="t('notification.type')"
                  :hint="t('notification.customTypeHint')"
                  persistent-hint
                  active
                  prepend-inner-icon="mdi-cog"
                />
              </VCol>
              <VCol cols="12" md="6">
                <VTextField
                  v-model="notificationInfo.name"
                  :label="t('notification.name')"
                  :hint="t('notification.nameRequired')"
                  persistent-hint
                  prepend-inner-icon="mdi-label"
                />
              </VCol>
            </VRow>
          </VForm>
        </VCardText>
        <VCardActions class="pt-3">
          <VBtn @click="openTestMessageDialog" prepend-icon="mdi-send" color="primary" variant="tonal" class="px-5">
            {{ t('common.testSend') }}
          </VBtn>
          <VBtn @click="saveNotificationInfo" prepend-icon="mdi-content-save" class="px-5">
            {{ t('common.confirm') }}
          </VBtn>
        </VCardActions>
      </VCard>
    </VDialog>

    <!-- 测试消息对话框 -->
    <VDialog
      v-if="testMessageDialog"
      v-model="testMessageDialog"
      max-width="30rem"
      :fullscreen="!display.mdAndUp.value"
    >
      <VCard>
        <VCardItem class="py-2">
          <template #prepend>
            <VIcon icon="mdi-send" class="me-2" />
          </template>
          <VCardTitle>{{ t('common.testSend') }}</VCardTitle>
          <VCardSubtitle>{{ props.notification.name }}</VCardSubtitle>
        </VCardItem>
        <VDialogCloseBtn @click="testMessageDialog = false" />
        <VDivider />
        <VCardText>
          <VForm>
            <VRow>
              <VCol cols="12">
                <VTextField
                  v-model="testMessage.title"
                  :label="t('notification.title')"
                  placeholder="测试消息"
                  persistent-hint
                  prepend-inner-icon="mdi-format-title"
                />
              </VCol>
              <VCol cols="12">
                <VTextarea
                  v-model="testMessage.content"
                  :label="t('notification.content')"
                  placeholder="这是一条测试消息..."
                  rows="4"
                  persistent-hint
                  prepend-inner-icon="mdi-message-text"
                />
              </VCol>
            </VRow>
          </VForm>
        </VCardText>
        <VCardActions class="pt-3">
          <VBtn @click="testMessageDialog = false" color="secondary" variant="tonal" class="px-5">
            {{ t('common.cancel') }}
          </VBtn>
          <VBtn @click="sendTestMessage" prepend-icon="mdi-send" color="primary" class="px-5" :loading="sendingTestMessage">
            {{ t('common.send') }}
          </VBtn>
        </VCardActions>
      </VCard>
    </VDialog>
  </div>
</template>
