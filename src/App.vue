<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from "vue";
import useApiKey from "./AppState";
import ChatGPTAPI from "./ChatGPTAPI";

interface ChatModel {
  isGPT: boolean;
  message: string;
}

const { getApiKey, setApiKey } = useApiKey();
const showHelp = ref(false); // 展示帮助面板
const officialTips = ref(""); // 展示官方对话
const items = ref<ChatModel[]>([]);
const inputText = ref("");
const isLoading = ref(false);
const api = ref<ChatGPTAPI>(new ChatGPTAPI(""));

function renew() {
  const apiKey = getApiKey();
  if (apiKey != null && apiKey.length > 0) {
    api.value = new ChatGPTAPI(apiKey);
  }
}

renew();

function toggleHelp() {
  showHelp.value = !showHelp.value;
}

function newChat() {
  items.value = [];
  renew();
}

function addItem() {
  const input = inputText.value.trim();
  inputText.value = "";

  if (input.startsWith("key:")) {
    // 更新 key
    const key = input.replace("key:", "");

    setApiKey(key);
    renew();

    officialTips.value = "API Key 已更新。";
    return;
  }

  if (input == "登出") {
    officialTips.value = "API Key 已清空。";

    // 登出，清空 key
    setApiKey("");
    return;
  }

  const apiKey = getApiKey();
  if (apiKey == null || apiKey.length == 0) {
    officialTips.value =
      "请先按照格式「key:YOUR_API_KEY」格式输入以更新 API Key。";
    return;
  }

  // 有 key
  officialTips.value = "";

  if (isLoading.value) {
    // 加载中
    // 不应该出现这种情况，应该禁用
    inputText.value = input;
  } else {
    // 非加载中
    if (input) {
      isLoading.value = true;
      items.value.push({ isGPT: false, message: input });
      items.value.push({ isGPT: true, message: "💭 思考中..." });

      // 请求 ChatGPT
      api.value
        .sendMessageStream(input)
        .then((reply) => {
          isLoading.value = false;
          items.value[items.value.length - 1].message = reply.trim();
        })
        .catch((error) => {
          isLoading.value = false;
          items.value[items.value.length - 1] = {
            isGPT: true,
            message: `[出错咯]${error}`,
          };
        });
    }
  }
}

function stop() {
  isLoading.value = false;
}

// 输入框占位符
const inputPlaceholder = computed(() => {
  if (isLoading.value) {
    return "💭 思考中...";
  } else {
    return "请在此输入消息...";
  }
});

// 键盘与输入框
const inputBottom = ref(0);

function handleKeyboardEvent(event: any) {
  if (event.type === "show") {
    // 键盘弹起时，将input组件显示在键盘上方
    inputBottom.value = event.height;
  } else if (event.type === "hide") {
    // 键盘收起时，将input组件重新放置在屏幕底部
    inputBottom.value = 0;
  }
}

onMounted(() => {
  // 监听键盘事件
  window.addEventListener("keyboardWillShow", handleKeyboardEvent);
  window.addEventListener("keyboardWillHide", handleKeyboardEvent);
});

onUnmounted(() => {
  // 取消监听键盘事件
  window.removeEventListener("keyboardWillShow", handleKeyboardEvent);
  window.removeEventListener("keyboardWillHide", handleKeyboardEvent);
});
</script>

<template>
  <div class="chatgkd">
    <!-- Header -->
    <div class="header">
      <span class="button" @click="toggleHelp">帮助</span>
      <div class="title-container">
        <span class="title">ChatGKD</span>
        <a class="subtitle" href="https://github.com/kingcos/ChatGKD"
          >Powered by kingcos.me</a
        >
      </div>
      <span class="button" @click="newChat">新对话</span>
    </div>

    <!-- 帮助区域 -->
    <div class="help" v-if="showHelp">
      <div>
        1. 欢迎使用 ChatGKD for web，本项目已开源在
        <a href="https://github.com/kingcos/ChatGKD"
          >github.com/kingcos/ChatGKD</a
        >；<br />
        2. ChatGKD for web 仅做接口封装与页面展示，不提供 API Key 且不对 OpenAI
        内容负责，请自行申请使用并对内容负责；<br />
        3. API Key 将只保存在浏览器本地存储，不会做上传或其他操作；<br />
        4. 请按照格式「key:YOUR_API_KEY」格式输入，即可更新 API Key；<br />
        5. 请输入「登出」即可清除本地存储的 API Key；<br />
      </div>
    </div>
    <div class="list" v-if="items.length > 0 || officialTips">
      <div class="tips" v-if="officialTips">{{ officialTips }}</div>
      <div v-for="(item, index) in items" :key="index" class="item">
        <span v-if="item.isGPT">🤖️：</span>
        <span v-else>🧑：</span>
        <span v-if="item.isGPT">{{ item.message }}</span>
        <span v-else class="human-text">{{ item.message }}</span>
      </div>
    </div>

    <!-- Footer -->
    <div class="footer" :style="{ bottom: inputBottom + 'px' }">
      <input
        type="text"
        v-model="inputText"
        v-bind:placeholder="inputPlaceholder"
        v-bind:disabled="isLoading"
        @keydown.enter="addItem"
      />
      <button v-if="isLoading" class="footer-button" @click="stop">🛑</button>
      <button v-else class="footer-button" @click="addItem">发送</button>
    </div>
  </div>
</template>

<style scoped>
body {
  padding: 0px;
  margin: 0px;
}

.chatgkd {
  display: flex;
  flex-direction: column;
  height: 100%;

  padding-top: 50px;
  padding-left: 10px;
  padding-right: 10px;

  padding-bottom: 50px; /* 预留input组件的高度 */
}

.header {
  position: fixed;
  left: 0;
  right: 0;
  top: 0;
  height: 50px;

  display: flex;
  align-items: center;
  padding: 10px;
  background-color: #e2e2e2;
}

.title {
  font-size: 24px;
  font-weight: 600;
}

.subtitle {
  font-size: 12px;
}

.title-container {
  flex: 1;
  display: flex;
  align-items: center;
  flex-direction: column;
}

.button {
  color: blue;
  border: 1px solid #000;
  padding: 5px;
  border-radius: 5px;
}

.footer {
  position: fixed;
  left: 0;
  right: 0;
  height: 50px;
}

.help {
  position: fixed;
  left: 0;
  right: 0;
  top: 70px;

  background-color: rgba(83, 97, 246, 0.95);
  color: #ffffff;
  padding: 10px;
  margin: 10px;
  border-radius: 10px;
}

.list {
  flex: 1;
  overflow-y: auto;

  margin-top: 30px;
  margin-bottom: 10px;
  padding: 10px 10px 0px 10px;
  border-radius: 10px;
  background-color: #efefef;
}

.tips {
  padding-bottom: 10px;
}

.item {
  padding-bottom: 10px;
}

.human-text {
  font-weight: 600;
}

input {
  width: 100%;
  height: 100%;
  border: none;
  outline: none;
  font-size: 14px;
  padding: 0 10px;

  background-color: #e2e2e2;
}

.footer-button {
  position: fixed;

  right: 0;
  bottom: 0;

  width: 80px;
  height: 50px;

  font-size: 16px;
  padding-left: 10px;
  padding-right: 10px;
}
</style>
