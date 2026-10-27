<template>
  <v-row
    align="center"
    justify="space-between"
    class="mb-2"
  >
    <v-col cols="auto">
      <h2>{{ $t('pages.nodes') }}</h2>
    </v-col>
    <v-col
      cols="auto"
      class="d-flex ga-2"
    >
      <v-btn
        color="secondary"
        variant="outlined"
        :loading="syncing"
        @click="syncNodes"
      >
        {{ $t('nodes.sync') }}
      </v-btn>
      <v-btn
        color="primary"
        @click="showModal()"
      >
        {{ $t('actions.add') }}
      </v-btn>
    </v-col>
  </v-row>

  <v-alert
    type="info"
    variant="tonal"
    class="mb-4"
  >
    {{ $t('nodes.help') }}
  </v-alert>

  <v-row>
    <v-col
      cols="12"
      sm="6"
      lg="4"
      xl="3"
    >
      <NodeStatusCard
        :name="$t('nodes.local')"
        :host-name="localStatus?.sys.hostName"
        :address="localHost"
        icon="mdi-server-network"
        :status="$t('online')"
        status-color="success"
        :core-running="localStatus?.sbd.running ?? null"
        :inbounds="localInbounds.map(inbound => `${inbound.tag} (${inbound.type})`)"
        :version="localStatus?.sys.appVersion"
      />
    </v-col>
    <v-col
      v-for="node in nodes"
      :key="node.id"
      cols="12"
      sm="6"
      lg="4"
      xl="3"
    >
      <NodeStatusCard
        :name="node.name"
        :host-name="node.hostName"
        :address="node.apiUrl"
        icon="mdi-earth"
        v-bind="nodeStatus(node)"
        :core-running="node.online ? node.coreRunning : null"
        :inbounds="node.inboundTags"
        :version="node.appVersion"
        :last-seen-at="node.online ? 0 : node.lastSeenAt"
        :last-checked-at="node.lastCheckedAt"
        :error="node.lastError"
      >
        <template #actions>
          <v-btn
            variant="text"
            @click="showModal(node)"
          >
            {{ $t('actions.edit') }}
          </v-btn>
          <v-spacer />
          <v-btn
            color="warning"
            variant="text"
            @click="deleteNode(node)"
          >
            {{ $t('actions.del') }}
          </v-btn>
        </template>
      </NodeStatusCard>
    </v-col>
  </v-row>

  <v-dialog
    v-model="dialog"
    max-width="700"
  >
    <v-card
      :title="$t(isEdit ? 'nodes.editTitle' : 'nodes.addTitle')"
      rounded="lg"
    >
      <v-divider />
      <v-card-text>
        <v-text-field
          v-model="draft.name"
          :label="$t('nodes.name')"
        />
        <v-text-field
          v-model="draft.apiUrl"
          :label="$t('nodes.apiUrl')"
          dir="ltr"
        />
        <v-text-field
          v-model="draft.token"
          :label="$t('nodes.token')"
          :hint="$t('nodes.tokenHint', { admins: $t('pages.admins'), token: $t('admin.api.token'), add: $t('actions.add') })"
          type="password"
          autocomplete="new-password"
          dir="ltr"
          persistent-hint
        />
        <v-row
          align="center"
          class="mb-2"
        >
          <v-col>{{ $t('nodes.remoteInbounds') }}</v-col>
          <v-col cols="auto">
            <v-btn
              variant="outlined"
              :loading="probing"
              @click="probeNode"
            >
              {{ $t('nodes.test') }}
            </v-btn>
          </v-col>
        </v-row>
        <v-alert
          v-if="probeError"
          type="error"
          density="compact"
          variant="tonal"
          class="mb-3"
        >
          {{ probeError }}
        </v-alert>
        <v-checkbox
          v-for="inbound in remoteInbounds"
          :key="inbound.id"
          v-model="draft.inboundTags"
          :value="inbound.tag"
          :label="`${inbound.tag} (${inbound.type}:${inbound.listen_port})`"
          density="compact"
          hide-details
        />
        <v-alert
          v-if="remoteInbounds.length === 0"
          type="info"
          density="compact"
          variant="tonal"
        >
          {{ $t('nodes.testHint') }}
        </v-alert>
        <v-switch
          v-model="draft.enabled"
          :label="$t('nodes.enabled')"
          color="primary"
          hide-details
        />
      </v-card-text>
      <v-card-actions>
        <v-spacer />
        <v-btn
          variant="text"
          @click="dialog = false"
        >
          {{ $t('actions.close') }}
        </v-btn>
        <v-btn
          color="primary"
          :loading="saving"
          @click="saveNode"
        >
          {{ $t('actions.save') }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script lang="ts" setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import Data from '@/store/modules/data'
import HttpUtils from '@/plugins/httputil'
import { i18n } from '@/locales'
import NodeStatusCard from '@/components/NodeStatusCard.vue'

interface RemoteInbound {
  id: number
  type: string
  tag: string
  listen_port: number
}
interface ManagedNode {
  id: number
  name: string
  apiUrl: string
  token: string
  inboundTags: string[]
  subUri: string
  enabled: boolean
  online: boolean
  hostName: string
  appVersion: string
  coreRunning: boolean
  lastCheckedAt: number
  lastSeenAt: number
  lastError?: string
  lastSyncAt?: number
}

interface NodeStatus {
  sys: { hostName: string, appVersion: string }
  sbd: { running: boolean }
}

const nodes = ref<ManagedNode[]>([])
const localStatus = ref<NodeStatus | null>(null)
const remoteInbounds = ref<RemoteInbound[]>([])
const dialog = ref(false)
const isEdit = ref(false)
const probing = ref(false)
const saving = ref(false)
const syncing = ref(false)
const probeError = ref('')
const draft = ref<ManagedNode>(emptyNode())
const localHost = window.location.host
const localInbounds = computed(() => Data().inbounds ?? [])

function emptyNode(): ManagedNode {
  return {
    id: 0, name: '', apiUrl: '', token: '', inboundTags: [], subUri: '', enabled: true,
    online: false, hostName: '', appVersion: '', coreRunning: false, lastCheckedAt: 0, lastSeenAt: 0,
  }
}

function nodeStatus(node: ManagedNode) {
  if (!node.enabled) return { status: i18n.global.t('disable'), statusColor: 'grey' }
  if (node.lastCheckedAt === 0) return { status: i18n.global.t('nodes.checking'), statusColor: 'warning' }
  if (node.online) return { status: i18n.global.t('online'), statusColor: 'success' }
  return { status: i18n.global.t('nodes.offline'), statusColor: 'error' }
}

async function loadNodes() {
  const [nodesMsg, statusMsg] = await Promise.all([
    HttpUtils.get<{ nodes: ManagedNode[] }>('api/nodes'),
    HttpUtils.get<NodeStatus>('api/status', { r: 'sys,sbd' }),
  ])
  if (nodesMsg.success) nodes.value = nodesMsg.obj.nodes
  if (statusMsg.success) localStatus.value = statusMsg.obj
}

function showModal(node?: ManagedNode) {
  isEdit.value = !!node
  draft.value = node ? { ...node, inboundTags: [...node.inboundTags] } : emptyNode()
  remoteInbounds.value = []
  probeError.value = ''
  dialog.value = true
}

async function probeNode() {
  probing.value = true
  probeError.value = ''
  const msg = await HttpUtils.post<{ inbounds: RemoteInbound[], subUri: string }>('api/probeNode', {
    apiUrl: draft.value.apiUrl,
    token: draft.value.token,
  })
  if (msg.success) {
    remoteInbounds.value = msg.obj.inbounds ?? []
    draft.value.subUri = msg.obj.subUri ?? ''
    const available = new Set(remoteInbounds.value.map(inbound => inbound.tag))
    draft.value.inboundTags = draft.value.inboundTags.filter(tag => available.has(tag))
    if (draft.value.inboundTags.length === 0) {
      draft.value.inboundTags = remoteInbounds.value.map(inbound => inbound.tag)
    }
  } else {
    probeError.value = msg.msg
  }
  probing.value = false
}

async function saveNode() {
  saving.value = true
  const action = isEdit.value ? 'edit' : 'new'
  const success = await Data().save('nodes', action, draft.value)
  if (success) {
    dialog.value = false
    await loadNodes()
  }
  saving.value = false
}

async function deleteNode(node: ManagedNode) {
  if (!window.confirm(`${i18n.global.t('confirm')} ${node.name}?`)) return
  if (await Data().save('nodes', 'del', node.id)) await loadNodes()
}

async function syncNodes() {
  syncing.value = true
  await HttpUtils.post('api/syncNodes', {})
  await loadNodes()
  syncing.value = false
}

let intervalId: ReturnType<typeof setInterval> | null = null

onMounted(() => {
  loadNodes()
  intervalId = setInterval(loadNodes, 10000)
})

onBeforeUnmount(() => {
  if (intervalId) clearInterval(intervalId)
})
</script>
