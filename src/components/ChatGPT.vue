<script setup>
import micromark from "../utils/micromark";
import { debounce } from "lodash-es";
import { ref, computed, reactive, watch, onMounted, nextTick } from "vue";
import { post } from "../utils/request";
import { setItem, getItem } from "../utils/storage";

const key = ref("");

const keyDialog = ref(false);

const data = reactive([
  {
    role: "chatgpt",
    content: "你可以问我各种问题哦 , 右上角设置你的api key \n",
    meta: {
      questions: [
        "写一个文案",
        "hello world代码示例",
        "写一首诗",
        "润色一下刚才的话",
        "周末安排",
        "讲个笑话",
        "写一个100字的科幻故事",
      ],
    },
  },
]);

const loading = ref(false);

const chatPanelRef = ref();

const input = ref("");

const favs = reactive([]);
onMounted(async () => {
  async function initCofig() {
    const f = await getItem("favs", true);
    const array = (f && JSON.parse(f)) || [];
    array.forEach((fav) => {
      favs.push(fav);
    });
    const chatRecords0 = await getItem("chatRecord");
    const chatRecords = (chatRecords0 && JSON.parse(chatRecords0)) || [];
    if (chatRecords.length) {
      data.splice(0, 1);
      chatRecords.forEach((rec) => {
        data.push(rec);
      });
    }
    key.value = await getItem("chatGptApiKey", true);
    nextTick(scrollToBottom);
  }

  function watchConfig2Save() {
    const saveFavs = debounce((v) => {
      setItem({ favs: JSON.stringify(v) }, true);
    }, 1000);
    const saveKey = debounce((v) => {
      setItem({ chatGptApiKey: v }, true);
    }, 1000);
    const saveData = debounce((v) => {
      setItem({ chatRecord: JSON.stringify(v) });
    }, 1000);
    watch(favs, (v) => saveFavs(v));
    watch(key, (v) => saveKey(v));
    watch(data, (v) => saveData(v));
  }
  await initCofig();
  watchConfig2Save();
});

const repeat = (prompt) => {
  submitSend(prompt);
};

const copy = async (prompt) => {
  let d = document.createElement("p");
  d.innerHTML = marked(prompt);
  await navigator.clipboard.writeText(d.innerText);
};

const send = () => {
  const prompt = input.value;

  submitSend(prompt);
  input.value = "";
};

const addFav = (prompt) => {
  favs.push(prompt);
};

const delFav = (i) => {
  favs.splice(i, 1);
};

const submitSend = async (prompt) => {
  if (!prompt.trim().length) {
    return;
  }
  loading.value = true;
  prompt = prompt || input.value;
  data.push({
    role: "me",
    content: prompt,
  });
  scrollToBottom();
  try {
    await sending(prompt);
  } catch (e) {
    data.push({
      role: "chatgpt",
      content: "这会有点累了，等会再问吧，或者换一下右上角的chatgpt api key",
    });
    scrollToBottom();
  }
  loading.value = false;
};

const sending = async (prompt) => {
  const d = await post(
    "https://api.openai.com/v1/chat/completions",
    {
      messages: [{ role: "user", content: prompt }],
      model: "gpt-3.5-turbo",
    },
    {
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${key.value}`,
      },
    }
  );

  // console.log(d);
  const r = d.choices[0].message.content.trim();
  data.push({
    role: "chatgpt",
    content: "",
  });
  const j = data.length - 1;
  for (let i = 0; i < r.length; i++) {
    setTimeout(() => {
      data[j].content += r.charAt(i);
      // 滚动条
      if (i % 20 == 0 || i == r.length - 1) {
        scrollToBottom();
      }
    }, i * 20);
  }
};

const clearChatRecord = () => {
  data.splice(0, data.length);
};

const scrollToBottom = debounce(() => {
  const domWrapper = chatPanelRef.value;
  const currentScroll = domWrapper.scrollTop; // 已经被卷掉的高度
  const clientHeight = domWrapper.offsetHeight; // 容器高度
  const scrollHeight = domWrapper.scrollHeight; // 内容总高度
  if (scrollHeight - 10 > currentScroll + clientHeight) {
    domWrapper.scrollTo(0, scrollHeight - clientHeight);
  }
}, 100);
</script>

<template>
  <div class="chat-warp">
    <div class="chat-header">
      <div class="title">
        <span>ChatGPT</span>
      </div>
      <div v-if="keyDialog">
        <label for="">chatgpt api key: </label>
        <input
          class="input"
          type="text"
          v-model="key"
          placeholder="Please enter your chat gpt api key."
          @keypress.enter="keyDialog = false"
        />
      </div>

      <img
        v-else
        class="icon"
        src="../assets/settings.svg"
        @click="keyDialog = true"
      />
    </div>
    <div ref="chatPanelRef" class="chat-panel">
      <section class="chat-line-warp" v-for="(item, i) in data" :key="i">
        <section class="chat-line" v-if="item.role == 'chatgpt'">
          <span class="avatar"> <img src="../assets/openai.svg" alt="" /></span>
          <div class="msg">
            <div v-html="micromark(item.content)"></div>
            <ul class="question" v-if="item.meta && item.meta.questions">
              <li
                class="hover"
                v-for="i in item.meta.questions"
                :key="i"
                v-text="i"
                @click="repeat(i)"
              ></li>
            </ul>
          </div>
        </section>
        <div class="chat-line chat-line-user" v-else>
          <div class="msg" v-html="micromark(item.content)"></div>
          <span class="avatar">
            <img src="../assets/user.svg" alt="" />
          </span>
        </div>
        <div class="chat-line-action">
          <img
            src="../assets/copy.svg"
            title="copy"
            @click="copy(item.content)"
          />
          <img
            src="../assets/repeat.svg"
            title="repeat"
            @click="repeat(item.content)"
          />
          <img
            src="../assets/favor.svg"
            title="favor"
            @click="addFav(item.content)"
          />
        </div>
      </section>
      <section class="chat-line-warp" v-show="loading">
        <section class="chat-line">
          <span class="avatar"> <img src="../assets/openai.svg" alt="" /></span>
          <div class="msg"><p>请稍等， 正在思考中...</p></div>
        </section>
      </section>
      <div v-show="data.length" class="clear-warp">
        <button @click="clearChatRecord" class="clear">
          Clear chat record
        </button>
      </div>
    </div>
    <div class="chat-footer">
      <div class="favs">
        <ul class="question">
          <li v-for="(item, i) in favs" :key="i">
            <span class="hover" v-text="item" @click="input = item"></span>
            <img
              class="del"
              @click="delFav(i)"
              src="../assets/delete.svg"
              alt=""
            />
          </li>
        </ul>
      </div>
      <div class="chat-send-box">
        <input
          class="textarea"
          type="text"
          v-model="input"
          placeholder="Please enter your chat message and press enter to send."
          @keypress.enter="send()"
        />
        <button class="btn" :disabled="loading" @click="send()">
          <svg
            class="send"
            width="24"
            height="24"
            viewBox="0 0 24 24"
            fill="none"
            xmlns="http://www.w3.org/2000/svg"
          >
            <path
              d="M5.97828 3.04161L21.3189 11.1188C22.0312 11.4939 22.0311 12.5138 21.3187 12.8886L5.97774 20.961C5.21303 21.3634 4.33238 20.6719 4.54192 19.8336L6 14L13 12.004L6 10L4.54225 4.16899C4.33263 3.33054 5.21355 2.63896 5.97828 3.04161Z"
              :fill="input ? '#00A457' : '#ccc'"
            ></path>
          </svg>
        </button>
      </div>
    </div>
  </div>
</template>

<style lang="less" scoped>
.chat-warp {
  height: 100vh;
  // max-width: 1024px;
  margin: 0 auto;
  background: transparent;
  color: var(--text-color);

  display: grid;
  grid-template-rows: 40px 1fr 75px;
  font-size: 14px;
  .chat-panel {
    overflow: auto;
  }
  .chat-header,
  .chat-panel {
    padding: var(--padding);
    box-sizing: border-box;
  }
  .chat-header {
    background: var(--msg-bg-color);

    border-top-right-radius: var(--gap-padding);
    border-top-left-radius: var(--gap-padding);
  }
  .chat-panel {
    background: var(--bg-color);
  }
  .chat-footer {
    background: var(--bg-color);
    padding: 3px var(--padding);
    // border-bottom-right-radius: var(--gap-padding);
    // border-bottom-left-radius: var(--gap-padding);
  }
}
.logo-warp {
  width: 40px;
  height: 40px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: var(--gap-padding);
  background-image: -webkit-linear-gradient(
    bottom right,
    #00c157,
    #00b08c,
    #009eb7,
    #00c157
  );
}
.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  .icon {
    width: 20px;
    height: 20px;
    cursor: pointer;
  }
  .title {
    display: flex;
    .logo {
      width: 30px;
      height: 30px;
    }
    span {
      font-size: 1.3em;
      color: var(--text-color);
      opacity: 0.7;
      // display: inline-block;
      // margin-left: 10px;
      // background-image: -webkit-linear-gradient(
      //   right,
      //   #00c157,
      //   #00b08c,
      //   #009eb7
      // );
      // -webkit-background-clip: text;
      // -webkit-text-fill-color: transparent;
    }
  }
}
.chat-panel {
  // border-bottom: 1px solid var(--msg-bg-color);
  .chat-line {
    // background: rgba(0, 0, 0, 0.1);
    // border: 1px solid #ccc;
    display: grid;
    grid-template-columns: 36px auto;
    grid-gap: var(--gap-padding);
    padding: 10px;
    border-radius: 10px;
    margin-bottom: 10px;
  }
  .chat-line-user {
    grid-template-columns: auto 36px;
    text-align: right;
    .avatar {
      background: #00a457;
    }
  }
  .chat-line-user ~ .chat-line-action {
    left: var(--gap-padding);
  }
  .avatar {
    display: block;
    width: 36px;
    height: 36px;
    line-height: 36px;
    background: #666;
    border-radius: var(--gap-padding);
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    img {
      width: 26px;
      height: 26px;
    }
  }
  .msg {
    // border: 1px solid #444;
    border-radius: 10px;
    padding: var(--gap-padding);
    background: var(--msg-bg-color);
  }
  .chat-line-warp {
    position: relative;
    &:hover .chat-line-action {
      display: grid;
    }
    .chat-line-action {
      display: none;
      width: 80px;
      background: rgba(0, 0, 0, 0.5);
      padding: 10px;
      position: absolute;
      top: var(--gap-padding);
      right: var(--gap-padding);
      border-radius: var(--gap-padding);
      grid-template-columns: repeat(3, 1fr);
      grid-gap: var(--gap-padding);
      img {
        width: 15px;
        height: 15px;
        opacity: 0.9;
        cursor: pointer;
      }
    }
  }
}

.chat-avatar {
  color: #009eb7;
}
.chat-footer {
  position: relative;
  .favs {
    position: absolute;
    bottom: 65px;
    ul {
      list-style: none;
      padding-inline-start: 0;
      li {
        position: relative;
        display: inline;
        border: 1px solid #ccc;
        padding: var(--gap-padding);
        border-radius: var(--gap-padding);
        background: var(--msg-bg-color);
        margin-right: var(--gap-padding);
        .del {
          // top: -8px;
          // position: absolute;
          width: 10px;
          height: 10px;
          cursor: pointer;
        }
      }
    }
  }
}
.chat-send-box {
  padding: 10px;
  display: grid;
  grid-template-columns: 1fr 30px;
  grid-gap: var(--gap-padding);
  border-radius: var(--gap-padding);
  box-shadow: 3px 3px 10px var(--msg-bg-color),
    -3px -3px 10px var(--msg-bg-color);
  // border: 1px solid #eee;
  .textarea {
    color: var(--text-color);
    border: 0;
    padding: 0.5em;
    border-radius: 10px;
    // border: 1px solid #ccc;
    background: var(--bg-color);

    // background: rgba(255, 255, 255, 0.6);
    &:focus {
      outline: var(--bg-color);
    }
    &:hover {
      outline: var(--bg-color);
    }
  }
  .btn {
    padding: 0.3em;
    background: transparent;
    .send {
      color: #fff;
    }
  }
}
.question {
  list-style: none;
  padding-inline-start: 0px;
  .hover {
    display: inline;
    padding: var(--gap-padding);
    &:hover {
      cursor: pointer;
      text-decoration: underline;
    }
  }
}
.input {
  border: 0;
  padding: 0.5em;
  border-radius: 10px;
  border: 1px solid #ccc;
  background: rgba(255, 255, 255, 0.6);
  &:focus {
    outline-color: rgba(200, 200, 200, 0.6);
  }
}
.clear-warp {
  margin: 0 auto;
  text-align: center;
  button {
    color: rgba(255, 100, 100, 1);
    background: var(--msg-bg-color);
  }
}
</style>
