<script setup>
import { NInput, NButton, NCard, NSpin, NDatePicker, NSelect, NFormItem, NInputNumber, NLayout, NMenu, NAlert } from 'naive-ui'
import { watch, ref } from "vue";
import MarkdownIt from 'markdown-it';
import { fetchEventSource, EventStreamContentType } from '@microsoft/fetch-event-source';
import { useStorage } from '@vueuse/core';
import { useIsMobile } from '../utils/composables'

import { MENU_OPTIONS, DIVINATION_OPTIONS, ABOUT } from "../config/constants";
const isMobile = useIsMobile()

const state_jwt = useStorage('jwt')
const prompt = ref("");
const result = useStorage("result", "");
const tmp_result = ref("");
const prompt_type = useStorage("prompt_type", "tarot");
const menu_type = useStorage("menu_type", "divination");
const lunarBirthday = useStorage("lunarBirthday", '龙年 庚辰年 七月十八 巨蟹座')
const birthday = useStorage("birthday", "2000-08-17 00:00:00");
const loading = ref(false);
const API_BASE = import.meta.env.VITE_API_BASE || "";
const md = new MarkdownIt();
const about = md.render(ABOUT);
const sex = ref("")
const surname = ref("")
const new_name_prompt = ref("")
const sexOptions = [
  { label: "男", value: "男" },
  { label: "女", value: "女" },
]
const plum_flower = useStorage("plum_flower", { num1: 0, num2: 0 })
const fate_body = useStorage("fate_body", { name1: "", name2: "" })

const onSubmit = async () => {
  try {
    loading.value = true;
    tmp_result.value = "";
    await fetchEventSource(`${API_BASE}/api/divination`, {
      method: "POST",
      body: JSON.stringify({
        prompt: prompt.value || "我的财务状况如何",
        prompt_type: prompt_type.value,
        birthday: birthday.value,
        new_name: {
          surname: surname.value,
          sex: sex.value,
          birthday: birthday.value,
          new_name_prompt: new_name_prompt.value
        },
        plum_flower: prompt_type.value == "plum_flower" ? plum_flower.value : null,
        fate: prompt_type.value == "fate" ? fate_body.value : null
      }),
      headers: {
        "Authorization": `Bearer ${state_jwt.value || "xxx"}`,
        "Content-Type": "application/json"
      },
      async onopen(response) {
        if (response.ok && response.headers.get('content-type') === EventStreamContentType) {
          return;
        } else if (response.status >= 400 && response.status < 500 && response.status !== 429) {
          throw new Error(`占卜失败: ${response.status}`);
        }
      },
      onmessage(msg) {
        if (msg.event === 'FatalError') {
          throw new FatalError(msg.data);
        }
        if (!msg.data) {
          return;
        }
        try {
          tmp_result.value += JSON.parse(msg.data);
          result.value = md.render(tmp_result.value);
        } catch (error) {
          console.error(error);
        }
      },
      onclose() {

      },
      onerror(err) {
        result.value = `占卜失败: ${err.message}`;
        throw new Error(`占卜失败: ${err.message}`);
      }
    });
  } catch (error) {
    console.error(error);
    result.value = error.message || "占卜失败";
  } finally {
    loading.value = false;
  }
};

watch(birthday, async (newBirthday, oldBirthday) => {
  lunarBirthday.value = '转换中...'
  try {
    const res = await fetch(`${API_BASE}/api/date?date=${newBirthday}`)
    lunarBirthday.value = await res.json()
  } catch (error) {
    console.error(error)
  }
})
</script>

<template>
  <div>
    <n-spin size="large" description="正在占卜..." :show="loading">
      <n-menu accordion v-model:value="prompt_type" :mode="isMobile ? 'vertical' : 'horizontal'"
        :options="MENU_OPTIONS" />
      <n-card v-if="prompt_type != 'about'">
        <div style="display: inline-block;">
          <n-form-item label="当前占卜" label-placement="left">
            <n-select v-model:value="prompt_type" :consistent-menu-width="false" value-field="key"
              :options="DIVINATION_OPTIONS" disabled />
          </n-form-item>
        </div>
        <div v-if="prompt_type == 'tarot'">
          <n-input v-model:value="prompt" type="textarea" round maxlength="40" :autosize="{ minRows: 3 }"
            placeholder="我的财务状况如何" />
        </div>
        <div v-if="prompt_type == 'birthday'">
          <div style="display: inline-block;">
            <n-form-item label="生日" label-placement="left">
              <n-date-picker v-model:formatted-value="birthday" value-format="yyyy-MM-dd HH:mm:ss" type="datetime" />
            </n-form-item>
            <p>农历: {{ lunarBirthday }}</p>
          </div>
        </div>
        <div v-if="prompt_type == 'new_name'">
          <div style="display: inline-block;">
            <n-form-item label="姓氏" label-placement="left">
              <n-input v-model:value="surname" type="text" maxlength="2" placeholder="请输入姓氏" />
            </n-form-item>
            <n-form-item label="性别" label-placement="left">
              <n-select v-model:value="sex" :options="sexOptions" />
            </n-form-item>
            <n-form-item label="生日" label-placement="left">
              <n-date-picker v-model:formatted-value="birthday" value-format="yyyy-MM-dd HH:mm:ss" type="datetime" />
            </n-form-item>
            <n-form-item label="附加" label-placement="left">
              <n-input v-model:value="new_name_prompt" type="text" maxlength="20" placeholder="" />
            </n-form-item>
            <p>农历: {{ lunarBirthday }}</p>
          </div>
        </div>
        <div v-if="prompt_type == 'name'">
          <div style="display: inline-block;">
            <n-input v-model:value="prompt" type="text" maxlength="10" round placeholder="请输入姓名" />
          </div>
        </div>
        <div v-if="prompt_type == 'dream'">
          <n-input v-model:value="prompt" type="textarea" round maxlength="40" :autosize="{ minRows: 3 }"
            placeholder="请输入你的梦境" />
        </div>
        <div v-if="prompt_type == 'plum_flower'">
          <div style="display: inline-block;">
            <h4>请随机输入两个 0-1000 的数字</h4>
            <n-form-item label="数字一" label-placement="left">
              <n-input-number v-model:value="plum_flower.num1" :min="0" :max="1000" />
            </n-form-item>
            <n-form-item label="数字二" label-placement="left">
              <n-input-number v-model:value="plum_flower.num2" :min="0" :max="1000" />
            </n-form-item>
          </div>
        </div>
        <div v-if="prompt_type == 'fate'">
          <div style="display: inline-block;">
            <h4>缘分是天定的，幸福是自己的。</h4>
            <p>想知道你和 ta 有没有缘分呢，编辑“姓名1” “姓名2”，然后点击“一键预测”。</p>
            <p>如郭靖 黄蓉，然后点击一键预测。 就能查看你和 ta 的缘分了。</p>
            <n-form-item label="姓名1" label-placement="left">
              <n-input v-model:value="fate_body.name1" round maxlength="40" />
            </n-form-item>
            <n-form-item label="姓名2" label-placement="left">
              <n-input v-model:value="fate_body.name2" round maxlength="40" />
            </n-form-item>
            <div class="footer" style="text-align:center">
              
            </div>
          </div>
        </div>
        <div v-if="menu_type != 'about'" class="button-container">
          <n-button class="center" @click="onSubmit" tertiary round type="primary">
            占卜
          </n-button>
        </div>
      </n-card>
    </n-spin>
    <n-card :title="prompt_type == 'about' ? '' : '占卜结果'">
      <div v-if="prompt_type != 'about'" class="result" v-html="result"></div>
      <div v-else class="result" v-html="about"></div>
    </n-card>
  </div>
</template>

<style scoped>
.button-container {
  display: flex;
  justify-content: center;
}

.n-button {
  margin-top: 12px;
  text-align: center;
  margin-bottom: 12px;
}

.result {
  text-align: left;
}
</style>
