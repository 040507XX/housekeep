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
      <!-- 左侧会话列表 -->
      <div class="session-list" v-if="showSessionList">
        <div class="session-header">
          <h3>聊天列表</h3>
        </div>
        <el-scrollbar>
          <div 
            v-for="session in sessions" 
            :key="session.id"
            class="session-item"
            :class="{ 
              active: currentSession?.id === session.id,
              'has-unread': session.unreadCount > 0 && session.lastSenderId !== userId
            }"
          >
            <div class="session-content" @click="selectSession(session)">
              <el-avatar :size="40" :src="getSessionAvatar(session)">
                {{ getSessionName(session)?.charAt(0) || '?' }}
              </el-avatar>
              <div class="session-info">
                <div class="session-name">
                  {{ getSessionName(session) }}
                  <span v-if="session.unreadCount > 0 && session.lastSenderId !== userId" class="unread-indicator"></span>
                </div>
                <div class="last-message">
                  {{ session.lastMessage || '暂无消息' }}
                  <!-- <span v-if="session.lastSenderId === userId" class="message-status">
                    {{ getMessageStatus(session) }}
                  </span> -->
                </div>
              </div>
              <el-badge 
                v-if="session.unreadCount > 0 && session.lastSenderId !== userId" 
                :value="session.unreadCount" 
                class="unread-badge"
              />
            </div>
            <div class="session-actions" v-if="!isStaff">
              <el-popconfirm
                title="确定要删除这个会话吗？"
                @confirm="deleteSession(session.id)"
                confirm-button-text="确定"
                cancel-button-text="取消"
              >
                <template #reference>
                  <el-icon class="delete-icon"><Delete /></el-icon>
                </template>
              </el-popconfirm>
            </div>
          </div>
        </el-scrollbar>
      </div>

      <!-- 右侧聊天区域 -->
      <div class="chat-main">
        <div class="chat-header">
          <span>{{ currentChatName }}</span>
          <div class="header-actions" v-if="messages.length && !isStaff">
            <el-button type="danger" size="small" @click="confirmClearMessages">
              清空聊天记录
            </el-button>
          </div>
        </div>
        
        <!-- 消息列表 -->
        <div class="message-list" ref="messageList">
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
                  :src="msg.senderId === userId ? userAvatar : targetAvatar"
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
    default: null
  },
  showSessionList: {
    type: Boolean,
    default: true
  }
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
const sessions = ref([])
const currentSession = ref(null)
const messages = ref([])
const inputMessage = ref('')
const ws = ref(null)
const messageList = ref(null)

// 计算属性
const dialogVisible = computed({
  get: () => props.modelValue,
  set: (val) => emit('update:modelValue', val)
})

const title = computed(() => props.showSessionList ? '消息中心' : '在线咨询')

const currentChatName = computed(() => {
  if (!currentSession.value) return ''
  return getSessionName(currentSession.value)
})

const targetName = computed(() => {
  if (!currentSession.value) return ''
  return userStore.userInfo?.roleCode === 'STAFF' 
    ? currentSession.value.user?.name 
    : currentSession.value.staff?.user?.name
})

const targetAvatar = computed(() => {
  if (!currentSession.value) return ''
  if (userStore.userInfo?.roleCode === 'STAFF') {
    const avatar = currentSession.value.user?.avatar
    return avatar ? `/api${avatar}` : ''
  }
  return ''
})

// 添加一个计算属性来判断当前用户是否是服务人员
const isStaff = computed(() => userStore.userInfo?.roleCode === 'STAFF')

// 方法
const getSessionName = (session) => {
  if (!session) return '未知用户';
  
  if (userStore.userInfo?.roleCode === 'STAFF') {
    return session.user?.name || session.user?.username || '未知用户1';
  } else {
    return session.staff?.user?.name || session.staff?.user?.username || '未知用户2';
  }
}

const getSessionAvatar = (session) => {
  if (!session) return '';
  
  if (userStore.userInfo?.roleCode === 'STAFF') {
    const avatar = session.user?.avatar
    return avatar ? `/api${avatar}` : '';
  } else {
    return '';
  }
}

const formatTime = (time) => {
  return formatDate(time, 'HH:mm')
}

// WebSocket连接
const connectWebSocket = () => {
    console.log("userId.value", userId.value)
    // 使用相对路径，避免硬编码主机名和端口
    const wsProtocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:'
    const wsUrl = `${wsProtocol}//localhost:1234/websocket/chat/${userId.value}`
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
    
    ws.value.onerror = (error) => {
        console.error('WebSocket连接错误:', error)
        ElMessage.error('聊天连接失败，请刷新页面重试')
    }
}

// 加载会话列表
const loadSessions = async () => {
  try {
    const params = {
      userId: userId.value,
      pageNum: 1,
      pageSize: 20
    }
    await request.get('/chat/sessions', params, {
      showDefaultMsg: false,
      onSuccess: (data) => {
        sessions.value = data.records || []
        // 如果指定了目标ID,选择对应会话
        if (props.targetId) {
          const targetSession = sessions.value.find(s => 
            userStore.userInfo?.roleCode === 'STAFF' 
              ? s.userId === props.targetId
              : s.staffId === props.targetId
          )
          if (targetSession) {
            selectSession(targetSession)
          } else {
            // 如果没有找到对应会话，但有targetId，创建新会话
            createNewSession()
          }
        } else if (sessions.value.length > 0) {
          selectSession(sessions.value[0])
        }
      },
      onError: (error) => {
        console.error('加载会话列表失败:', error)
      }
    })
  } catch (error) {
    console.error('加载会话列表失败:', error)
  }
}

// 加载聊天记录
const loadMessages = async (sessionId) => {
  try {
    await request.get('/chat/messages', {
      sessionId,
      pageNum: 1,
      pageSize: 50
    }, {
      showDefaultMsg: false,
      onSuccess: (data) => {
        messages.value = data.records || []
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

// 选择会话
const selectSession = async (session) => {
  currentSession.value = session
  await loadMessages(session.id)
  // 标记消息为已读
  if (session.unreadCount > 0) {
    markMessagesAsRead(session.id)
  }
}

// 发送消息
const sendMessage = async () => {
  if (!inputMessage.value?.trim()) {
    return
  }
  
  // 检查是否有当前会话
  if (!currentSession.value) {
    ElMessage.warning('请先选择聊天对象')
    return
  }
  
  const content = inputMessage.value.trim()
  const message = {
    receiverId: userStore.userInfo?.roleCode === 'STAFF' 
      ? currentSession.value?.userId
      : currentSession.value?.staff?.userId,
    content: content
  }
  
  // 检查接收者ID是否存在
  if (!message.receiverId) {
    ElMessage.warning('无法确定消息接收者')
    return
  }
  
  console.log('发送消息:', message)
  ws.value.send(JSON.stringify(message))
  
  // 立即在本地添加消息
  const localMessage = {
    id: Date.now(), // 临时ID
    sessionId: currentSession.value.id,
    senderId: userId.value,
    receiverId: message.receiverId,
    content: content,
    createTime: new Date().toISOString(),
    readStatus: 0,
    messageType: 1
  }
  messages.value.push(localMessage)
  
  // 更新会话的最后一条消息
  currentSession.value.lastMessage = content
  
  // 滚动到底部
  nextTick(() => {
    scrollToBottom()
  })
  
  inputMessage.value = ''
}

// 处理新消息
const handleNewMessage = (message) => {
  // 如果没有当前会话，但收到了消息，需要重新加载会话列表
  if (!currentSession.value) {
    loadSessions()
  }
  messages.value.push(message)
  nextTick(() => {
    scrollToBottom()
  })
}

// 标记消息已读
const markMessagesAsRead = async (sessionId) => {
  try {
    await request.post('/chat/messages/read?sessionId=' + sessionId + '&userId=' + userId.value, null, {
      showDefaultMsg: false,
      onSuccess: () => {
        const session = sessions.value.find(s => s.id === sessionId)
        if (session) {
          session.unreadCount = 0
        }
      },
      onError: (error) => {
        console.error('标记消息已读失败:', error)
      }
    })
  } catch (error) {
    console.error('标记消息已读失败:', error)
  }
}

// 滚动到底部
const scrollToBottom = () => {
  const container = messageList.value?.querySelector('.el-scrollbar__wrap')
  if (container) {
    container.scrollTop = container.scrollHeight
  }
}

// 创建新会话
const createNewSession = async () => {
  try {
    const data = {
      userId: userStore.userInfo?.roleCode === 'STAFF' ? props.targetId : userId.value,
      staffId: userStore.userInfo?.roleCode === 'STAFF' ? userId.value : props.targetId
    }
    console.log("创建会话参数:", data)
    await request.post('/chat/sessions?userId=' + userId.value + '&staffId=' + props.targetId, null, {
      showDefaultMsg: false,
      onSuccess: (data) => {
        // 创建成功后重新加载会话列表
        loadSessions()
      },
      onError: (error) => {
        console.error('创建会话失败:', error)
        ElMessage.error('创建会话失败')
      }
    })
  } catch (error) {
    console.error('创建会话失败:', error)
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
  if (!currentSession.value) return
  
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
  if (!currentSession.value) return
  
  try {
    await request.delete(`/chat/sessions/${currentSession.value.id}/messages`, null, {
      successMsg: '聊天记录已清空',
      onSuccess: () => {
        messages.value = []
        // 更新会话的最后一条消息
        currentSession.value.lastMessage = ''
      }
    })
  } catch (error) {
    console.error('清空聊天记录失败:', error)
  }
}

// 删除会话
const deleteSession = async (sessionId) => {
  try {
    await request.delete(`/chat/sessions/${sessionId}`, null, {
      successMsg: '会话已删除',
      onSuccess: () => {
        // 从会话列表中移除
        sessions.value = sessions.value.filter(s => s.id !== sessionId)
        
        // 如果删除的是当前会话，则清空当前会话和消息
        if (currentSession.value && currentSession.value.id === sessionId) {
          currentSession.value = null
          messages.value = []
        }
        
        // 如果还有其他会话，选择第一个
        if (sessions.value.length > 0 && !currentSession.value) {
          selectSession(sessions.value[0])
        }
      }
    })
  } catch (error) {
    console.error('删除会话失败:', error)
  }
}

// 获取消息状态
const getMessageStatus = (session) => {
  // 如果最后发送者是当前用户，显示已读/未读状态
  if (session.lastSenderId === userId.value) {
    return session.unreadCount > 0 ? '未读' : '已读';
  }
  return '';
}

// 生命周期钩子
onMounted(async () => {
  if (dialogVisible.value) {
    await loadSessions()
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
    await loadSessions()
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
  display: flex;
  height: 600px;
  
  .session-list {
    width: 280px;
    border-right: 1px solid #e0e0e0;
    display: flex;
    flex-direction: column;
    
    .session-header {
      padding: 15px;
      border-bottom: 1px solid #e0e0e0;
      
      h3 {
        margin: 0;
        font-size: 16px;
      }
    }
    
    .session-item {
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 12px;
      cursor: pointer;
      transition: all 0.3s;
      
      &:hover {
        background-color: #f5f7fa;
      }
      
      &.active {
        background-color: #ecf5ff;
      }
      
      &.has-unread {
        background-color: rgba(64, 158, 255, 0.1);
        font-weight: bold;
      }
      
      .session-content {
        display: flex;
        align-items: center;
        flex: 1;
      }
      
      .session-actions {
        opacity: 0;
        transition: opacity 0.3s;
        
        .delete-icon {
          color: #999;
          font-size: 16px;
          
          &:hover {
            color: #f56c6c;
          }
        }
      }
      
      &:hover .session-actions {
        opacity: 1;
      }
      
      .session-info {
        flex: 1;
        margin-left: 12px;
        overflow: hidden;
        
        .session-name {
          font-size: 14px;
          margin-bottom: 4px;
          
          .unread-indicator {
            display: inline-block;
            width: 8px;
            height: 8px;
            border-radius: 50%;
            background-color: #f56c6c;
            margin-left: 5px;
          }
        }
        
        .last-message {
          font-size: 12px;
          color: #999;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          
          .message-status {
            font-size: 12px;
            color: #909399;
            margin-left: 5px;
          }
        }
      }
      
      .unread-badge {
        margin-left: 8px;
      }
    }
  }
  
  .chat-main {
    flex: 1;
    display: flex;
    flex-direction: column;
    
    .chat-header {
      padding: 15px;
      border-bottom: 1px solid #e0e0e0;
      font-size: 16px;
      font-weight: 500;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    
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
            display: flex;
            align-items: center;
            justify-content: space-between;
            font-size: 12px;
            color: #999;
            margin-top: 4px;
            
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
}
</style> 