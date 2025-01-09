<template>
  <div class="chat-container">
    <!-- 左侧历史对话窗口 -->
    <div class="histrory-area">
      <div class="histrory-header">
        <h2>历史对话</h2>
        <el-button type="primary" @click="newConversation">新建对话</el-button>
      </div>
      <div class="conversation-list">
        <el-scrollbar>
          <el-menu :default-active="currentConversationId" @select="handleConversationSelect">
            <template v-for="conversation in conversations" :key="conversation.id">
              <el-menu-item :index="conversation.id">
                <span>对话 {{ formatDate(conversation.createdAt) }}</span>
                <el-button type="danger" size="small" @click.stop="deleteConversation(conversation.id)">删除</el-button>
              </el-menu-item>
            </template>
          </el-menu>
        </el-scrollbar>
      </div>
    </div>

    <!-- 右侧当前对话窗口 -->
    <div class="chat-area">
      <div class="message-list">
        <el-scrollbar>
          <div v-for="(message, index) in messages" :key="index" :class="['message', message.role]">
            <div class="message-content">{{ message.content }}</div>
          </div>
        </el-scrollbar>
      </div>
      <div class="message-input-container">
        <el-input v-model="inputMessage" class="message-input" placeholder="输入消息..." @keyup.enter="sendMessage"
          :disabled="isLoading">
          <template #append>
            <el-button type="primary" @click="sendMessage" :disabled="!inputMessage.trim() || isLoading"
              :loading="isLoading">
              发送
            </el-button>
          </template>
        </el-input>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, nextTick } from 'vue'

interface Message {
  role: 'user' | 'assistant'
  content: string
  timestamp: number
}

interface Conversation {
  id: string
  messages: Message[]
  createdAt: number
}

const conversations = ref<Conversation[]>([])
const currentConversationId = ref<string | null>(null)
const messages = computed(() => {
  const conversation = conversations.value.find(c => c.id === currentConversationId.value)
  return conversation ? conversation.messages : []
})

const inputMessage = ref('')
const isLoading = ref(false)

const newConversation = () => {
  const newId = Date.now().toString()
  conversations.value.push({
    id: newId,
    messages: [],
    createdAt: Date.now()
  })
  currentConversationId.value = newId
  nextTick(() => {
    const conversationList = document.querySelector('.conversation-list .el-scrollbar__wrap');
    if (conversationList) {
      conversationList.scrollTop = conversationList.scrollHeight;
    }
  })
}

const handleConversationSelect = (id: string) => {
  currentConversationId.value = id
}

const formatDate = (timestamp: number) => {
  return new Date(timestamp).toLocaleTimeString()
}

const deleteConversation = (id: string) => {
  conversations.value = conversations.value.filter(c => c.id !== id);
  if (currentConversationId.value === id) {
    currentConversationId.value = conversations.value[0]?.id || null;
  }
}

async function sendMessage() {
  if (!inputMessage.value.trim() || isLoading.value) return;

  const userMessage = inputMessage.value.trim();
  inputMessage.value = '';

  if (!currentConversationId.value) {
    newConversation();
  }

  const currentConversation = conversations.value.find(c => c.id === currentConversationId.value);
  if (!currentConversation) return;

  // 添加用户消息到当前对话
  currentConversation.messages.push({
    role: 'user',
    content: userMessage,
    timestamp: Date.now()
  });

  isLoading.value = true;

  try {
    // 构建消息历史
    const chatHistory = currentConversation.messages.map(msg => ({
      role: msg.role,
      content: msg.content
    }));

    const response = await fetch('http://localhost:11434/api/chat', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        model: 'qwen2.5-coder:7b',
        messages: chatHistory,
        stream: true,
      }),
    });

    if (!response.ok) {
      throw new Error('API 请求失败');
    }

    const reader = response.body?.getReader();
    if (!reader) throw new Error('无法获取响应读取器');

    const decoder = new TextDecoder();
    let assistantMessage = '';

    // 添加 AI 消息占位符到当前对话
    currentConversation.messages.push({
      role: 'assistant',
      content: '',
      timestamp: Date.now()
    });

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const chunk = decoder.decode(value);
      const lines = chunk.split('\n');

      for (const line of lines) {
        if (!line.trim()) continue;
        try {
          const data = JSON.parse(line);
          if (data.message?.content) {
            assistantMessage += data.message.content;
            // 更新当前对话中的最后一条消息
            currentConversation.messages[currentConversation.messages.length - 1].content = assistantMessage;
            // 滚动到底部位置
            nextTick(() => {
              const messageList = document.querySelector('.message-list .el-scrollbar__wrap');
              if (messageList) {
                messageList.scrollTop = messageList.scrollHeight;
              }
            });
          }
        } catch (error) {
          console.error('解析错误:', error);
        }
      }
    }
  } catch (error) {
    console.error('API 调用失败:', error);
    currentConversation.messages.push({
      role: 'assistant',
      content: '抱歉，请求失败，请稍后再试',
      timestamp: Date.now()
    });
  } finally {
    isLoading.value = false;
  }
}
</script>

<style scoped>
.chat-container {
  display: flex;
  height: 100vh;
  overflow: hidden;
}

.histrory-area {
  width: 300px;
  background-color: #f5f7fa;
  border-right: 1px solid #e4e7ed;
  display: flex;
  flex-direction: column;
}

.histrory-header {
  padding: 5px;
  background-color: #409EFF;
  color: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.conversation-list {
  flex: 1;
  overflow: hidden;
}

.chat-area {
  flex: 1;
  display: flex;
  flex-direction: column;
  background-color: #fafafa;
}

.message-list {
  flex: 1;
  padding: 20px;
  overflow: hidden;
}

.message-input-container {
  padding: 20px;
  background-color: white;
  border-top: 1px solid #e4e7ed;
}

.message {
  margin-bottom: 10px;
}

.message-content {
  max-width: 70%;
  padding: 12px 16px;
  border-radius: 12px;
  line-height: 1.5;
  word-wrap: break-word;
  white-space: pre-wrap;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
  transition: all 0.2s ease;
}

.user {
  margin-left: auto;
}

.user .message-content {
  background-color: #409EFF;
  color: white;
  margin-left: auto;
}

.assistant {
  margin-right: auto;
}

.assistant .message-content {
  background-color: #f0f0f0;
  color: #333;
  margin-right: auto;
}

.message-input {
  width: 100%;
}
</style>