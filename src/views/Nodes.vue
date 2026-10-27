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
      md="4"
      lg="3"
    >
      <v-card
        rounded="xl"
        elevation="5"
        height="100%"
      >
        <v-card-title>{{ $t('nodes.local') }}</v-card-title>
        <v-card-subtitle>{{ localHost }}</v-card-subtitle>
        <v-card-text>
          <div
            v-for="inbound in localInbounds"
            :key="inbound.id"
            class="mb-1"
          >
            {{ inbound.tag }} <span class="text-medium-emphasis">({{ inbound.type }})</span>
          </div>
          <div v-if="localInbounds.length === 0">
            {{ $t('noData') }}
          </div>
        </v-card-text>
      </v-card>
    </v-col>
    <v-col
      v-for="node in nodes"
      :key="node.id"
      cols="12"
      sm="6"
      md="4"
      lg="3"
    >
      <v-card
        rounded="xl"
        elevation="5"
        height="100%"
      >
        <v-card-title class="d-flex align-center ga-2">
          {{ node.name }}
          <v-chip
            size="small"
            :color="node.enabled ? 'success' : 'default'"
          >
            {{ node.enabled ? $t('enable') : $t('disable') }}
          </v-chip>
        </v-card-title>
        <v-card-subtitle class="text-break">
          {{ node.apiUrl }}
        </v-card-subtitle>
        <v-card-text>
          <div
            v-for="tag in node.inboundTags"
            :key="tag"
            class="mb-1"
          >
            {{ tag }}
          </div>
          <v-alert
            v-if="node.lastError"
            type="warning"
            density="compact"
            variant="tonal"
            class="mt-2"
          >
            {{ node.lastError }}
          </v-alert>
        </v-card-text>
        <v-divider />
        <v-card-actions>
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
        </v-card-actions>
      </v-card>
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
          type="password"
          autocomplete="new-password"
          dir="ltr"
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
import { computed, onMounted, ref } from 'vue'
import Data from '@/store/modules/data'
import HttpUtils from '@/plugins/httputil'
import { i18n } from '@/locales'

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
  lastError?: string
  lastSyncAt?: number
}

const nodes = ref<ManagedNode[]>([])
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
  return { id: 0, name: '', apiUrl: '', token: '', inboundTags: [], subUri: '', enabled: true }
}

async function loadNodes() {
  const msg = await HttpUtils.get<{ nodes: ManagedNode[] }>('api/nodes')
  if (msg.success) nodes.value = msg.obj.nodes ?? []
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

onMounted(loadNodes)
</script>
