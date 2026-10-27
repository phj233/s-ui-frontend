<template>
  <v-card
    class="node-status-card d-flex flex-column"
    rounded="xl"
    elevation="2"
    height="100%"
  >
    <v-sheet
      :color="statusColor"
      height="4"
    />
    <v-card-text class="pa-5 pb-4 flex-grow-0">
      <div class="d-flex align-start ga-3">
        <v-avatar
          :color="statusColor"
          variant="tonal"
          rounded="lg"
          size="48"
        >
          <v-icon
            :icon="icon"
            size="26"
          />
        </v-avatar>
        <div class="node-status-card__identity">
          <div
            class="text-h6 font-weight-bold text-truncate"
            :title="name"
          >
            {{ name }}
          </div>
          <div
            class="text-body-2 text-medium-emphasis text-truncate"
            :title="hostName || address"
          >
            {{ hostName || address }}
          </div>
        </div>
      </div>
      <div class="d-flex align-center justify-space-between ga-2 mt-4">
        <v-chip
          :color="statusColor"
          variant="tonal"
          size="small"
          prepend-icon="mdi-circle-small"
        >
          {{ status }}
        </v-chip>
        <span
          v-if="version"
          class="text-caption text-medium-emphasis text-truncate"
          :title="`${$t('nodes.appVersion')}: ${version}`"
        >
          {{ version }}
        </span>
      </div>
      <div class="node-status-card__address d-flex align-center ga-2 mt-4 text-body-2 text-medium-emphasis">
        <v-icon
          icon="mdi-web"
          size="16"
        />
        <span
          class="text-truncate"
          :title="address"
        >{{ address }}</span>
      </div>
    </v-card-text>

    <v-card-text class="px-5 pt-0 pb-4 flex-grow-0">
      <div class="d-flex ga-2">
        <div class="node-status-card__metric">
          <div class="text-caption text-medium-emphasis">
            {{ $t('nodes.core') }}
          </div>
          <div class="d-flex align-center ga-1 mt-1 font-weight-medium">
            <v-icon
              :icon="coreRunning === null ? 'mdi-help-circle-outline' : coreRunning ? 'mdi-check-circle-outline' : 'mdi-close-circle-outline'"
              :color="coreRunning === null ? 'grey' : coreRunning ? 'success' : 'error'"
              size="18"
            />
            {{ coreRunning === null ? '—' : $t(coreRunning ? 'nodes.running' : 'nodes.stopped') }}
          </div>
        </div>
        <div class="node-status-card__metric">
          <div class="text-caption text-medium-emphasis">
            {{ $t('pages.inbounds') }}
          </div>
          <div class="text-h6 font-weight-bold mt-1">
            {{ inbounds.length }}
          </div>
        </div>
      </div>
      <div class="text-caption text-medium-emphasis mt-4 mb-2">
        {{ $t('pages.inbounds') }}
      </div>
      <div class="d-flex flex-wrap ga-1">
        <v-chip
          v-for="tag in inbounds"
          :key="tag"
          size="small"
          variant="outlined"
          label
        >
          {{ tag }}
        </v-chip>
        <span
          v-if="inbounds.length === 0"
          class="text-body-2 text-medium-emphasis"
        >{{ $t('noData') }}</span>
      </div>
      <v-alert
        v-if="error"
        type="warning"
        density="compact"
        variant="tonal"
        class="mt-4"
      >
        {{ error }}
      </v-alert>
    </v-card-text>

    <div
      v-if="lastSeenAt || lastCheckedAt"
      class="node-status-card__footer mt-auto px-5 py-3 text-caption text-medium-emphasis"
    >
      <div v-if="lastSeenAt">
        {{ $t('nodes.lastSeen') }} · {{ formatTime(lastSeenAt) }}
      </div>
      <div v-if="lastCheckedAt">
        {{ $t('nodes.lastChecked') }} · {{ formatTime(lastCheckedAt) }}
      </div>
    </div>
    <template v-if="$slots.actions">
      <v-divider />
      <v-card-actions class="px-3">
        <slot name="actions" />
      </v-card-actions>
    </template>
  </v-card>
</template>

<script lang="ts" setup>
defineProps<{
  name: string
  hostName?: string
  address: string
  icon: string
  status: string
  statusColor: string
  coreRunning: boolean | null
  inbounds: string[]
  version?: string
  lastSeenAt?: number
  lastCheckedAt?: number
  error?: string
}>()

function formatTime(value: number): string {
  return new Date(value * 1000).toLocaleString()
}
</script>

<style scoped>
.node-status-card {
  border: thin solid rgba(var(--v-border-color), var(--v-border-opacity));
}

.node-status-card__identity,
.node-status-card__address span {
  min-width: 0;
}

.node-status-card__metric {
  flex: 1;
  min-width: 0;
  padding: 12px;
  border-radius: 12px;
  background: rgba(var(--v-theme-on-surface), 0.05);
}

.node-status-card__footer {
  border-top: thin solid rgba(var(--v-border-color), var(--v-border-opacity));
}
</style>
