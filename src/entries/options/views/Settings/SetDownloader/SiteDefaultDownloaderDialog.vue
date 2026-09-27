<script setup lang="ts">
import { computed, ref } from "vue";
import { useI18n } from "vue-i18n";

import { useMetadataStore } from "@/options/stores/metadata.ts";
import { useConfigStore } from "@/options/stores/config.ts";
import type { IDefaultDownloaderConfig, TDownloaderKey } from "@/shared/types.ts";
import { getDownloaderIcon } from "@ptd/downloader";

import SiteFavicon from "@/options/components/SiteFavicon/Index.vue";

const showDialog = defineModel<boolean>();

const { t } = useI18n();
const metadataStore = useMetadataStore();
const configStore = useConfigStore();

/** 站点列表（与站点过滤对话框保持一致的口径） */
const addedSites = computed(() =>
  Object.entries(metadataStore.sites)
    .filter(([id, site]) => (configStore.contentScript.allowExceptionSites ? (site.allowContentScript ?? true) : true))
    .map(([id]) => ({ id, name: metadataStore.siteNameMap[id] ?? id }))
    .sort((a, b) => a.name.localeCompare(b.name)),
);

const enabledDownloaders = computed(() => metadataStore.getSortedEnabledDownloaders);

/** 弹窗内的本地草稿，确认时才写回 store */
const draftConfigs = ref<Record<string, IDefaultDownloaderConfig>>({});

const globalDefaultName = computed(() => {
  const downloader = metadataStore.defaultDownloader?.id
    ? metadataStore.downloaders[metadataStore.defaultDownloader.id]
    : undefined;
  return downloader ? `${downloader.name} [${downloader.address}]` : "";
});

const getConfig = (siteId: string): IDefaultDownloaderConfig => draftConfigs.value[siteId] ?? {};

const suggestFolders = (downloaderId?: TDownloaderKey): string[] =>
  (downloaderId ? metadataStore.downloaders[downloaderId]?.suggestFolders : undefined) ?? [];

const suggestTags = (downloaderId?: TDownloaderKey): string[] =>
  (downloaderId ? metadataStore.downloaders[downloaderId]?.suggestTags : undefined) ?? [];

function enterDialog() {
  const configs: Record<string, IDefaultDownloaderConfig> = {};
  for (const site of addedSites.value) {
    configs[site.id] = { ...(metadataStore.siteDefaultDownloaders?.[site.id] ?? {}) };
  }
  draftConfigs.value = configs;
}

/** 切换下载器时清空路径与标签（避免沿用上一个下载器的取值） */
function setDownloader(siteId: string, downloaderId?: TDownloaderKey) {
  draftConfigs.value[siteId] = {
    id: downloaderId,
    folder: getConfig(siteId).id === downloaderId ? (getConfig(siteId).folder ?? "") : "",
    tags: getConfig(siteId).id === downloaderId ? (getConfig(siteId).tags ?? "") : "",
  };
}

function setField(siteId: string, key: "folder" | "tags", value: string) {
  draftConfigs.value[siteId] = { ...getConfig(siteId), [key]: value };
}

function save() {
  metadataStore.setSiteDefaultDownloaders(draftConfigs.value);
  showDialog.value = false;
}
</script>

<template>
  <v-dialog v-model="showDialog" scrollable max-width="1000" @after-enter="enterDialog">
    <v-card>
      <v-card-title class="pa-0">
        <v-toolbar :title="t('SetDownloader.siteDefault.title')" color="blue-grey-darken-2">
          <template #append>
            <v-btn icon="mdi-close" :title="t('common.dialog.close')" @click="showDialog = false" />
          </template>
        </v-toolbar>
      </v-card-title>
      <v-divider />
      <v-card-text>
        <v-alert class="mb-3 py-2" color="info" variant="tonal">
          <template #text>
            {{ t("SetDownloader.siteDefault.hint") }}
          </template>
        </v-alert>

        <v-alert v-if="enabledDownloaders.length === 0" type="warning" variant="tonal">
          {{ t("SetDownloader.siteDefault.noEnabledDownloader") }}
        </v-alert>

        <v-skeleton-loader v-else-if="addedSites.length === 0" type="image" />

        <v-list v-else class="overflow-x-hidden overflow-y-hidden px-3 pt-3">
          <!-- Vuetify 4 的 v-row 默认 gap 为 24px，卡片间距由 v-col 的 pa-1 控制，需显式 gap="0" -->
          <v-row gap="0">
            <v-col v-for="site in addedSites" :key="site.id" cols="12" md="6" class="pa-1">
              <v-list-item border class="bg-grey-lighten-4">
                <template #prepend>
                  <SiteFavicon :site-id="site.id" class="mr-2" flush-on-click />
                </template>

                <template #title>
                  <v-list-item-title>
                    <b>{{ site.name }}</b>
                  </v-list-item-title>
                </template>

                <template #subtitle>
                  <v-chip label size="x-small" class="mt-1">
                    {{ site.id }}
                  </v-chip>
                </template>

                <div class="mt-2">
                  <v-select
                    :model-value="getConfig(site.id).id"
                    :items="enabledDownloaders"
                    :placeholder="t('SetDownloader.siteDefault.useGlobalDefault')"
                    clearable
                    density="compact"
                    hide-details
                    item-title="name"
                    item-value="id"
                    variant="outlined"
                    @update:model-value="(v) => setDownloader(site.id, v as TDownloaderKey)"
                  >
                    <template #item="{ props, item: downloader }">
                      <v-list-item
                        v-bind="props"
                        :prepend-avatar="getDownloaderIcon(downloader.type)"
                        :subtitle="downloader.address"
                        :title="downloader.name"
                      />
                    </template>
                  </v-select>

                  <template v-if="getConfig(site.id).id">
                    <v-combobox
                      :model-value="getConfig(site.id).folder"
                      :items="suggestFolders(getConfig(site.id).id)"
                      class="mt-2"
                      density="compact"
                      hide-details
                      :label="t('SetDownloader.PathAndTag.downloadPath.title')"
                      variant="outlined"
                      @update:model-value="(v) => setField(site.id, 'folder', (v as string) ?? '')"
                    />
                    <v-combobox
                      :model-value="getConfig(site.id).tags"
                      :items="suggestTags(getConfig(site.id).id)"
                      class="mt-2"
                      density="compact"
                      hide-details
                      :label="t('SetDownloader.PathAndTag.tags.title')"
                      variant="outlined"
                      @update:model-value="(v) => setField(site.id, 'tags', (v as string) ?? '')"
                    />
                  </template>
                </div>
              </v-list-item>
            </v-col>
          </v-row>
        </v-list>
      </v-card-text>
      <v-divider />
      <v-card-actions>
        <span v-if="globalDefaultName" class="text-caption text-medium-emphasis">
          {{ t("SetDownloader.siteDefault.globalDefaultTip") }}{{ globalDefaultName }}
        </span>
        <v-spacer />
        <v-btn color="error" prepend-icon="mdi-close-circle" variant="text" @click="showDialog = false">
          {{ t("common.dialog.cancel") }}
        </v-btn>
        <v-btn color="success" prepend-icon="mdi-check-circle-outline" variant="text" @click="save">
          {{ t("common.dialog.ok") }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<style scoped lang="scss"></style>
