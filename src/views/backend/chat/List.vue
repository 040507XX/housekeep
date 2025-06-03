<template>
  <div class="chat-list">
    <!-- 搜索和操作区域 -->
    <el-card class="action-card">
      <div class="action-header">
        <div class="left">
          <h2 class="page-title">聊天记录管理</h2>
          <el-tag type="info" effect="plain" class="chat-count">
            {{ total }} 条聊天记录
          </el-tag>
        </div>
        <div class="right">
          <el-input
            v-model="searchForm.content"
            placeholder="搜索聊天内容"
            clearable
            class="search-input"
            @keyup.enter="handleSearch"
          >
            <template #prefix>
              <el-icon><Search /></el-icon>
            </template>
          </el-input>
          <el-button :icon="Refresh" circle @click="handleRefresh" />
        </div>
      </div>

      <!-- 表格区域 -->
      <el-table 
        v-loading="loading" 
        :data="tableData" 
        border
      >
        <el-table-column prop="id" label="消息ID" width="80" />
        <el-table-column prop="sessionId" label="会话ID" width="80" />
        <el-table-column label="发送者" width="150">
          <template #default="{ row }">
            <div class="user-info">
              <el-avatar :size="24" :src="fileUtils.getImageUrl(row.sender?.avatar)">
                {{ row.sender?.name?.charAt(0) || '?' }}
              </el-avatar>
              <span>{{ row.sender?.name }}</span>
            </div>
          </template>
        </el-table-column>
        <el-table-column label="接收者" width="150">
          <template #default="{ row }">
            <div class="user-info">
              <el-avatar :size="24" :src="fileUtils.getImageUrl(row.receiver?.avatar)">
                {{ row.receiver?.name?.charAt(0) || '?' }}
              </el-avatar>
              <span>{{ row.receiver?.name }}</span>
            </div>
          </template>
        </el-table-column>
        <el-table-column prop="content" label="消息内容" min-width="250" show-overflow-tooltip>
          <template #default="{ row }">
            <div v-if="row.messageType === 1">{{ row.content }}</div>
            <el-image 
              v-else-if="row.messageType === 2"
              :src="row.content"
              :preview-src-list="[row.content]"
              fit="cover"
              class="message-image"
            />
            <div v-else class="system-message">{{ row.content }}</div>
          </template>
        </el-table-column>
        <el-table-column prop="messageType" label="消息类型" width="100">
          <template #default="{ row }">
            <el-tag :type="getMessageTypeTag(row.messageType)">
              {{ getMessageTypeText(row.messageType) }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="readStatus" label="状态" width="80">
          <template #default="{ row }">
            <el-tag :type="row.readStatus === 1 ? 'success' : 'warning'">
              {{ row.readStatus === 1 ? '已读' : '未读' }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="createTime" label="发送时间" width="160">
          <template #default="{ row }">
            {{ formatDate(row.createTime) }}
          </template>
        </el-table-column>
        <el-table-column label="操作" width="150" fixed="right">
          <template #default="{ row }">
            <el-button type="primary" link @click="handleView(row)">查看</el-button>
            <el-button type="danger" link @click="handleDelete(row)">删除</el-button>
          </template>
        </el-table-column>
      </el-table>

      <!-- 分页 -->
      <div class="pagination">
        <el-pagination
          v-model:current-page="currentPage"
          v-model:page-size="pageSize"
          :total="total"
          :page-sizes="[10, 20, 50, 100]"
          layout="total, sizes, prev, pager, next, jumper"
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
        />
      </div>
    </el-card>

    <!-- 消息详情对话框 -->
    <el-dialog
      title="消息详情"
      v-model="detailDialogVisible"
      width="600px"
    >
      <el-descriptions :column="1" border>
        <el-descriptions-item label="消息ID">{{ detail.id }}</el-descriptions-item>
        <el-descriptions-item label="会话ID">{{ detail.sessionId }}</el-descriptions-item>
        <el-descriptions-item label="发送者">{{ detail.sender?.name }}</el-descriptions-item>
        <el-descriptions-item label="接收者">{{ detail.receiver?.name }}</el-descriptions-item>
        <el-descriptions-item label="消息类型">{{ getMessageTypeText(detail.messageType) }}</el-descriptions-item>
        <el-descriptions-item label="消息内容">
          <div v-if="detail.messageType === 1">{{ detail.content }}</div>
          <el-image 
            v-else-if="detail.messageType === 2"
            :src="detail.content"
            fit="cover"
            style="max-width: 200px;"
          />
          <div v-else class="system-message">{{ detail.content }}</div>
        </el-descriptions-item>
        <el-descriptions-item label="读取状态">
          {{ detail.readStatus === 1 ? '已读' : '未读' }}
        </el-descriptions-item>
        <el-descriptions-item label="发送时间">{{ formatDate(detail.createTime) }}</el-descriptions-item>
      </el-descriptions>
    </el-dialog>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { formatDate } from '@/utils/dateUtils'
import request from '@/utils/request'
import { ElMessage, ElMessageBox } from 'element-plus'
import { Search, Refresh } from '@element-plus/icons-vue'
import fileUtils from '@/utils/fileUtils'  // 导入 fileUtils
// 数据列表
const tableData = ref([])
const loading = ref(false)
const total = ref(0)
const currentPage = ref(1)
const pageSize = ref(10)

// 搜索表单
const searchForm = ref({
  content: ''
})

// 详情对话框
const detailDialogVisible = ref(false)
const detail = ref({})

// 获取消息类型标签
const getMessageTypeTag = (type) => {
  const types = {
    1: '',  // 文本
    2: 'success', // 图片
    3: 'info'  // 系统消息
  }
  return types[type] || ''
}

// 获取消息类型文本
const getMessageTypeText = (type) => {
  const types = {
    1: '文本',
    2: '图片',
    3: '系统消息'
  }
  return types[type] || '未知'
}

// 查看消息详情
const handleView = (row) => {
  detail.value = row
  detailDialogVisible.value = true
}

// 获取聊天记录列表
const fetchChatMessages = async () => {
  loading.value = true
  try {
    const params = {
      pageNum: currentPage.value,
      pageSize: pageSize.value,
      content: searchForm.value.content
    }
    
    await request.get('/chat/messages/all', params, {
      showDefaultMsg: false,
      onSuccess: (data) => {
        tableData.value = data.records
        total.value = data.total
      }
    })
  } catch (error) {
    console.error('获取聊天记录失败:', error)
  } finally {
    loading.value = false
  }
}

// 删除聊天记录
const handleDelete = (row) => {
  ElMessageBox.confirm(
    '确定要删除这条聊天记录吗？此操作不可恢复。',
    '警告',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning'
    }
  ).then(async () => {
    try {
      await request.delete(`/chat/messages/${row.id}`, null, {
        successMsg: '删除成功'
      })
      fetchChatMessages()
    } catch (error) {
      console.error('删除聊天记录失败:', error)
    }
  })
}

// 搜索
const handleSearch = () => {
  currentPage.value = 1
  fetchChatMessages()
}

// 刷新
const handleRefresh = () => {
  searchForm.value.content = ''
  currentPage.value = 1
  fetchChatMessages()
  ElMessage.success('刷新成功')
}

// 分页大小改变
const handleSizeChange = (val) => {
  pageSize.value = val
  fetchChatMessages()
}

// 当前页改变
const handleCurrentChange = (val) => {
  currentPage.value = val
  fetchChatMessages()
}

onMounted(() => {
  fetchChatMessages()
})
</script>

<style lang="scss" scoped>
.chat-list {
  padding: 16px;
}

.action-card {
  background: #fff;
}

.action-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.page-title {
  font-size: 20px;
  font-weight: 600;
  color: #2c3e50;
  margin: 0;
}

.chat-count {
  font-size: 13px;
}

.right {
  display: flex;
  align-items: center;
  gap: 12px;
}

.search-input {
  width: 240px;
}

.user-info {
  display: flex;
  align-items: center;
  gap: 8px;
  
  .el-avatar {
    background: var(--el-color-primary);
  }
  
  span {
    font-size: 13px;
  }
}

// 分页样式
.pagination {
  margin-top: 16px;
  display: flex;
  justify-content: flex-end;
}

// 表格样式优化
:deep(.el-table) {
  .el-table__header-wrapper th {
    background: #f8fafc;
    color: #1e293b;
    font-weight: 600;
  }
  
  .el-table__row {
    .el-button {
      padding: 4px 8px;
    }
  }
}

.message-image {
  width: 60px;
  height: 60px;
  border-radius: 4px;
  cursor: pointer;
}

.system-message {
  color: #909399;
  font-style: italic;
}

// 详情对话框中的描述列表样式
:deep(.el-descriptions) {
  .el-descriptions__label {
    width: 120px;
    justify-content: flex-end;
  }
}
</style> 