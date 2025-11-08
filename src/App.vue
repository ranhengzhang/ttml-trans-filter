<script setup lang="ts">
import {computed, ref} from "vue";
// import { invoke } from "@tauri-apps/api/core";
import { NCheckbox, NCheckboxGroup, NSplit, NSpace, NInput, NButton, NIcon } from "naive-ui";
import { CopyOutline } from '@vicons/ionicons5'
import {writeText} from "@tauri-apps/plugin-clipboard-manager";

const selected_langs = ref<string[]>([]);
const source_ttml = ref("")

const show_button = ref(false)

// 计算属性：从source_ttml中提取语言标签并去重
const langs = computed(() => {
  if (!source_ttml.value) return [];

  // 使用正则匹配所有 xml:lang="xx-XX" 格式的字符串
  const langRegex = /(?<!<tt[^<>]+)xml:lang="([^"]+)"/g;
  const matches = source_ttml.value.matchAll(langRegex);

  // 提取匹配结果并去重
  const langSet = new Set<string>();
  for (const match of matches) {
    if (match[1]) langSet.add(match[1]);
  }

  return Array.from(langSet);
});
// 生成目标TTML（正则方案）
const target_ttml = computed(() => {
  if (!source_ttml.value) {
    return source_ttml.value;
  }

  if (source_ttml.value.indexOf("iTunesMetadata") != -1) {
    // 构造正则：匹配翻译行并捕获整个标签
    const translationRegex = /(<(translation|transliteration)(\s+type="(subtitle|replacement)")?\s+xml:lang="([^"]+)"[^>]*>(<text for="L\d+">[\s\S]*?<\/text>)+<\/(translation|transliteration)>)/g;

    return source_ttml.value.replace(translationRegex, (_, fullTag, _tag, _attr, _type, lang) => {
      // 保留选中的语言或非翻译行
      return selected_langs.value.includes(lang) ? fullTag : "";
    });
  } else {
    // 构造正则：匹配翻译行并捕获整个标签
    const translationRegex = /(<span\s+ttm:role="x-translation"\s+xml:lang="([^"]+)"[^>]*>[\s\S]*?<\/span>)/g;

    return source_ttml.value.replace(translationRegex, (_, fullTag, lang) => {
      // 保留选中的语言或非翻译行
      return selected_langs.value.includes(lang) ? fullTag : "";
    });
  }
});
</script>

<template>
  <n-space id="main" vertical size="large">
    <div id="lang">
      <n-checkbox-group v-model:value="selected_langs">
        可选语言：
        <n-checkbox v-for="lang in langs" :key="lang" :value="lang" :label="lang"/>
      </n-checkbox-group>
    </div>
    <n-split direction="horizontal" style="height: 100%" :max="0.75" :min="0.25">
      <template #1>
        <div id="source">
          <n-input
              v-model:value="source_ttml"
              type="textarea"
              placeholder="粘贴 TTML 到这里"
              autosize
          />
        </div>
      </template>
      <template #2>
        <div id="target" style="position: relative;"
             @mouseleave="show_button = false"
             @mouseover="show_button = true">
          <n-input
              v-model:value="target_ttml"
              type="textarea"
              placeholder="粘贴 TTML 到这里"
              autosize
          />
          <n-button id="copy" v-if="show_button" type="primary" size="tiny" @click="writeText(target_ttml)" text>
            <template #icon>
              <n-icon>
                <CopyOutline/>
              </n-icon>
            </template>
          </n-button>
        </div>
      </template>
    </n-split>
  </n-space>
</template>

<style>
* {
  padding: 0;
  margin: 0;
  font-size: 20px;
}

html, body, #app, #main > div:not(:first-child) {
  height: 100%;
  width: 100%;
}
#main {
  width: calc(100% - 40px);
  height: calc(100% - 100px);
  padding: 20px;
  justify-content: space-between;
}
#lang {
  line-height: 1.4em;
}
#source, #target, :is(#source, #target) .n-input {
  height: 100%;
}
:is(#source, #target) textarea {
  font-size: 16px;
  font-family: "Sarasa Mono SC", monospace;
  font-weight: 700;
  overflow-wrap: break-word;
  word-break: break-all;
  word-wrap: break-word;
  white-space: pre-wrap;
  height: 100%;
}
#copy {
  position: absolute;
  bottom: 10px;
  right: 10px;
}
</style>