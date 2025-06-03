<template>
  <el-dialog
    v-model="dialogVisible"
    :title="title"
    width="800px"
    :close-on-click-modal="false"
    destroy-on-close
    class="chat-dialog"
  >
    <div class="chat-container">
      <!-- 消息列表 -->
      <div class="message-list" ref="messageList">
        <div class="message-actions" v-if="!isStaff">
          <el-button type="danger" size="small" @click="confirmClearMessages" :disabled="!messages.length">
            清空聊天记录
          </el-button>
        </div>
        <el-scrollbar>
          <div class="messages-container">
            <div 
              v-for="msg in messages" 
              :key="msg.id"
              class="message-item"
              :class="{ 'message-mine': msg.senderId === userId }"
            >
              <el-avatar 
                :size="36" 
                :src="msg.senderId === userId ? userAvatar : ''"
              >
                {{ msg.senderId === userId ? (userName?.charAt(0) || '?') : (targetName?.charAt(0) || '?') }}
              </el-avatar>
              <div class="message-content">
                <div class="message-text">{{ msg.content }}</div>
                <div class="message-time">
                  {{ formatTime(msg.createTime) }}
                  <el-popconfirm
                    v-if="!isStaff && msg.senderId === userId"
                    title="确定要删除这条消息吗？"
                    @confirm="deleteMessage(msg.id)"
                    confirm-button-text="确定"
                    cancel-button-text="取消"
                  >
                    <template #reference>
                      <el-icon class="delete-icon"><Delete /></el-icon>
                    </template>
                  </el-popconfirm>
                </div>
              </div>
            </div>
          </div>
        </el-scrollbar>
      </div>

      <!-- 输入区域 -->
      <div class="chat-input">
        <el-input
          v-model="inputMessage"
          type="textarea"
          :rows="3"
          placeholder="请输入消息"
          @keyup.enter="sendMessage"
          @keydown.enter.prevent
        />
        <el-button type="primary" @click="sendMessage">发送</el-button>
      </div>
    </div>
  </el-dialog>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watch } from 'vue'
import { useUserStore } from '@/store/user'
import { formatDate } from '@/utils/dateUtils'
import request from '@/utils/request'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Delete } from '@element-plus/icons-vue'

const props = defineProps({
  modelValue: Boolean,
  targetId: {
    type: [String, Number],
    required: true
  },
  targetName: {
    type: String,
    required: true
  },
  targetAvatar: String
})

const emit = defineEmits(['update:modelValue'])

const userStore = useUserStore()
const userId = computed(() => userStore.userInfo?.id)
const userName = computed(() => userStore.userInfo?.name || userStore.userInfo?.username)
const userAvatar = computed(() => {
  const avatar = userStore.userInfo?.avatar
  return avatar ? `/api${avatar}` : ''
})

// 聊天相关数据
const messages = ref([])
const inputMessage = ref('')
const ws = ref(null)
const messageList = ref(null)
const sessionId = ref(null)

// 计算属性
const dialogVisible = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})

const title = computed(() => `与 ${props.targetName} 聊天`)

// 添加一个计算属性来判断当前用户是否是服务人员
const isStaff = computed(() => userStore.userInfo?.roleCode === 'STAFF')

// WebSocket连接
const connectWebSocket = () => {
  const wsUrl = `ws://localhost:1234/api/websocket/chat/${userId.value}`
  ws.value = new WebSocket(wsUrl)
  
  ws.value.onopen = () => {
    console.log('WebSocket连接已建立')
  }
  
  ws.value.onmessage = (event) => {
    const message = JSON.parse(event.data)
    handleNewMessage(message)
  }
  
  ws.value.onclose = () => {
    console.log('WebSocket连接已关闭')
  }
}

// 获取或创建会话
const getOrCreateSession = async () => {
  try {
  
    const res = await request.get('/chat/sessions', {
      userId: userId.value,
      targetId: props.targetId,
      pageNum: 1,
      pageSize: 1
    }, {
      showDefaultMsg: false,
      onSuccess: (data) => {
        if (data.records?.length > 0) {
          sessionId.value = data.records[0].id
        }
      },
      onError: (error) => {
        console.error('获取会话列表失败:', error)
      }
    })
    
    if (!sessionId.value) {
      // 如果没有会话,创建新会话
      await request.post('/chat/sessions?userId=' + userId.value + '&targetId=' + props.targetId, null, {
        showDefaultMsg: false,
        onSuccess: (data) => {
          sessionId.value = data.id
        },
        onError: (error) => {
          console.error('创建会话失败:', error)
        }
      })
    }
  } catch (error) {
    console.error('获取会话失败:', error)
  }
}

// 加载聊天记录
const loadMessages = async () => {
  if (!sessionId.value) return
  
  try {
    await request.get('/chat/messages', {
      sessionId: sessionId.value,
      pageNum: 1,
      pageSize: 50
    }, {
      showDefaultMsg: false,
      onSuccess: (data) => {
        messages.value = (data.records || []).reverse()
        nextTick(() => {
          scrollToBottom()
        })
      },
      onError: (error) => {
        console.error('加载聊天记录失败:', error)
      }
    })
  } catch (error) {
    console.error('加载聊天记录失败:', error)
  }
}

// 发送消息
const sendMessage = async () => {
  if (!inputMessage.value?.trim()) {
    return
  }
  
  const message = {
    receiverId: props.targetId,
    content: inputMessage.value.trim()
  }
  
  ws.value.send(JSON.stringify(message))
  inputMessage.value = ''
}

// 处理新消息
const handleNewMessage = (message) => {
  messages.value.push(message)
  nextTick(() => {
    scrollToBottom()
  })
}

// 标记消息已读
const markMessagesAsRead = async () => {
  if (!sessionId.value) return
  
  try {
    await request.post('/chat/messages/read', {
      sessionId: sessionId.value,
      userId: userId.value
    }, {
      showDefaultMsg: false,
      onSuccess: () => {
        console.log('消息已标记为已读')
      },
      onError: (error) => {
        console.error('标记消息已读失败:', error)
      }
    })
  } catch (error) {
    console.error('标记消息已读失败:', error)
  }
}

// 格式化时间
const formatTime = (time) => {
  return formatDate(time, 'HH:mm')
}

// 滚动到底部
const scrollToBottom = () => {
  const container = messageList.value?.querySelector('.el-scrollbar__wrap')
  if (container) {
    container.scrollTop = container.scrollHeight
  }
}

// 删除单条消息
const deleteMessage = async (messageId) => {
  try {
    await request.delete(`/chat/messages/${messageId}`, null, {
      successMsg: '删除成功',
      onSuccess: () => {
        // 从本地消息列表中移除
        messages.value = messages.value.filter(msg => msg.id !== messageId)
      }
    })
  } catch (error) {
    console.error('删除消息失败:', error)
  }
}

// 确认清空聊天记录
const confirmClearMessages = () => {
  if (!sessionId.value) return
  
  ElMessageBox.confirm(
    '确定要清空所有聊天记录吗？此操作不可恢复。',
    '警告',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning'
    }
  ).then(() => {
    clearMessages()
  }).catch(() => {
    // 用户取消操作
  })
}

// 清空聊天记录
const clearMessages = async () => {
  if (!sessionId.value) return
  
  try {
    await request.delete(`/chat/sessions/${sessionId.value}/messages`, null, {
      successMsg: '聊天记录已清空',
      onSuccess: () => {
        messages.value = []
      }
    })
  } catch (error) {
    console.error('清空聊天记录失败:', error)
  }
}

// 生命周期钩子
onMounted(async () => {
  if (dialogVisible.value) {
    await getOrCreateSession()
    await loadMessages()
    connectWebSocket()
  }
})

onUnmounted(() => {
  if (ws.value) {
    ws.value.close()
  }
})

// 监听对话框显示状态
watch(dialogVisible, async (val) => {
  if (val) {
    await getOrCreateSession()
    await loadMessages()
    connectWebSocket()
  } else {
    if (ws.value) {
      ws.value.close()
    }
  }
})
</script>

<style lang="scss" scoped>
.chat-dialog {
  :deep(.el-dialog__body) {
    padding: 0;
  }
}

.chat-container {
  height: 600px;
  display: flex;
  flex-direction: column;
  
  .message-list {
    flex: 1;
    padding: 20px;
    overflow-y: auto;
    
    .message-item {
      display: flex;
      margin-bottom: 20px;
      
      &.message-mine {
        flex-direction: row-reverse;
        
        .message-content {
          margin-left: 0;
          margin-right: 12px;
          
          .message-text {
            background-color: #95ec69;
          }
        }
      }
      
      .message-content {
        margin-left: 12px;
        max-width: 70%;
        
        .message-text {
          padding: 10px 15px;
          background-color: #f5f5f5;
          border-radius: 4px;
          word-break: break-all;
        }
        
        .message-time {
          font-size: 12px;
          color: #999;
          margin-top: 4px;
        }
      }
    }
  }
  
  .chat-input {
    padding: 15px;
    border-top: 1px solid #e0e0e0;
    display: flex;
    align-items: flex-end;
    gap: 10px;
    
    .el-input {
      flex: 1;
    }
    
    .el-button {
      height: 32px;
    }
  }
}

.message-actions {
  display: flex;
  justify-content: flex-end;
  padding: 0 0 10px 0;
}

.message-time {
  display: flex;
  align-items: center;
  justify-content: space-between;
  
  .delete-icon {
    cursor: pointer;
    color: #999;
    font-size: 14px;
    margin-left: 5px;
    opacity: 0;
    transition: opacity 0.3s;
    
    &:hover {
      color: #f56c6c;
    }
  }
  
  .read-status {
    font-size: 12px;
    color: #909399;
    margin-left: 5px;
  }
}

.message-item:hover {
  .delete-icon {
    opacity: 1;
  }
}
</style> 