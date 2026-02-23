<template>
  <view class="layout">

    <!-- 左侧功能栏 -->
    <view class="sidebar">

      <!-- 用户卡片 -->
      <view class="user-card">
        <view class="user-box">
          <view class="avatar">➖</view>
          <view class="info">
            <text class="name">班组扣款管理</text>
            <text class="role">扣款记录管理</text>
          </view>
          
          <!-- 返回主页 -->
          <view class="logout" @click="goBackToMain">
            返回主页
          </view>
        </view>
      </view>

      <!-- 菜单 -->
      <view class="menu">
        <view 
          class="menu-item" 
          :class="{active: currentView === 'add'}"
          @click="currentView = 'add'"
        >
          ➖ 扣款录入
        </view>
        
        <view 
          class="menu-item" 
          :class="{active: currentView === 'history'}"
          @click="loadHistory(); currentView = 'history'"
        >
          📋 历史记录
        </view>
      </view>

      <view class="footer">v0.0.1</view>
    </view>

    <!-- 主区 -->
    <view class="main">

      <view class="header">
        <text class="title">{{ currentView === 'add' ? '班组扣款管理' : '历史记录' }}</text>
      </view>

      <view class="content">
        <!-- 扣款录入 -->
        <view v-if="currentView === 'add'" class="card">
          <text class="card-title">扣款录入</text>

          <!-- 班组选择 -->
          <view class="form-section">
            <text class="section-title">选择班组 *</text>
            <view class="team-selector">
              <view 
                v-for="(teamItem, index) in teams" 
                :key="index"
                :class="['team-item', formData.team === teamItem ? 'active' : '']"
                @click="formData.team = teamItem"
              >
                {{ teamItem }}
              </view>
            </view>
          </view>

          <!-- 扣款类型 -->
          <view class="form-section">
            <text class="section-title">扣款类型 *</text>
            <view class="type-selector">
              <view 
                v-for="(type, index) in deductTypes" 
                :key="index"
                :class="['type-item', formData.deduct_type === type ? 'active' : '']"
                @click="formData.deduct_type = type"
              >
                {{ type }}
              </view>
            </view>
          </view>

          <!-- 金额输入 -->
          <view class="form-section">
            <text class="section-title">扣款金额 *</text>
            <view class="amount-input-wrapper">
              <text class="currency">¥</text>
              <input 
                v-model="formData.amount" 
                type="digit" 
                placeholder="请输入扣款金额" 
                class="amount-input"
              />
            </view>
          </view>

          <!-- 扣款原因 -->
          <view class="form-section">
            <text class="section-title">扣款原因（可选）</text>
            <textarea 
              v-model="formData.reason" 
              placeholder="请填写扣款的具体原因（默认为'无'）" 
              class="reason-textarea"
              maxlength="200"
            />
            <text class="char-count">{{ formData.reason.length }}/200</text>
          </view>

          <!-- 提交按钮 -->
          <view class="submit-section">
            <view class="summary-info">
              <text>班组：{{ formData.team || '未选择' }}</text>
              <text>类型：{{ formData.deduct_type || '未选择' }}</text>
              <text>金额：¥{{ formData.amount || '0.00' }}</text>
              <text>原因：{{ formData.reason || '无' }}</text>
            </view>
            
            <button 
              class="btn-primary" 
              @click="submitDeduct" 
              :disabled="!isFormValid || submitting"
            >
              {{ submitting ? '提交中...' : (isEditing ? '更新记录' : '确认扣款') }}
            </button>
            
            <button 
              class="btn-cancel" 
              @click="resetForm"
              :disabled="submitting"
            >
              {{ isEditing ? '取消编辑' : '清空重填' }}
            </button>
          </view>
        </view>

        <!-- 历史记录 -->
        <view v-if="currentView === 'history'" class="card">
          <text class="card-title">历史记录</text>
          
          <!-- 搜索和筛选区域 -->
          <view class="filter-bar">
            <input 
              v-model="searchKeyword" 
              class="search-input" 
              placeholder="搜索班组、类型..." 
              @input="filterHistory"
            />
            <picker :range="filterOptions" @change="e=>filterBy=e.detail.value">
              <view class="filter-picker">{{ filterBy || '全部' }}</view>
            </picker>
          </view>

          <view v-if="filteredList.length === 0" class="empty-tip">
            <text>暂无扣款记录</text>
          </view>

          <!-- 历史记录列表 -->
          <view v-for="item in paginatedList" :key="item.id" class="deduct-item">
            <view class="deduct-header">
              <text class="deduct-date">{{ formatDate(item.created_at) }}</text>
              <text class="deduct-amount">¥{{ item.amount }}</text>
            </view>
            
            <view class="deduct-details">
              <view class="detail-row">
                <text class="detail-label">班组：</text>
                <text class="detail-value">{{ item.team }}</text>
              </view>
              <view class="detail-row">
                <text class="detail-label">类型：</text>
                <text class="detail-value">{{ item.deduct_type }}</text>
              </view>
              <view class="detail-row">
                <text class="detail-label">原因：</text>
                <text class="detail-value">{{ item.reason || '无' }}</text>
              </view>
            </view>
            
            <view class="deduct-actions">
              <view class="action-btn edit-btn" @click="editDeduct(item)">
                <text>编辑</text>
              </view>
              <view class="action-btn delete-btn" @click="confirmDeleteDeduct(item)">
                <text>删除</text>
              </view>
            </view>
          </view>

          <!-- 分页控件 -->
          <view v-if="totalPages > 1" class="pagination">
            <view 
              class="pagination-btn" 
              :class="{disabled: currentPage === 1}"
              @click="goToPage(currentPage - 1)"
            >
              上一页
            </view>
            
            <view class="pagination-info">
              第 {{ currentPage }} 页 / 共 {{ totalPages }} 页
              <text class="pagination-total">（共 {{ filteredList.length }} 条记录）</text>
            </view>
            
            <view 
              class="pagination-btn" 
              :class="{disabled: currentPage === totalPages}"
              @click="goToPage(currentPage + 1)"
            >
              下一页
            </view>
          </view>

          <!-- 分页大小选择 -->
          <view v-if="filteredList.length > 0" class="page-size-selector">
            <text class="page-size-label">每页显示：</text>
            <picker :range="pageSizeOptions" @change="onPageSizeChange">
              <view class="page-size-picker">{{ pageSize }} 条</view>
            </picker>
          </view>
        </view>
      </view>
    </view>
    
    <!-- 删除确认弹窗 -->
    <view v-if="showDeleteConfirm" class="mask" @click="showDeleteConfirm = false">
      <view class="dialog" @click.stop>
        <text class="card-title">确认删除</text>
        <text class="delete-confirm-text">确定要删除这条扣款记录吗？</text>
        <text class="delete-record-info">班组：{{ deleteRecord.team }}</text>
        <text class="delete-record-info">金额：¥{{ deleteRecord.amount }}</text>
        
        <view class="dialog-buttons">
          <button class="btn-cancel" @click="showDeleteConfirm = false">取消</button>
          <button class="btn-primary" @click="deleteDeduct">确认删除</button>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

// 配置
const API_BASE = 'http://127.0.0.1/php/api2.php'

// 状态
const submitting = ref(false)
const currentView = ref('add') // 'add' 或 'history'
const historyList = ref([])
const filteredList = ref([])
const searchKeyword = ref('')
const filterBy = ref('')
const showDeleteConfirm = ref(false)
const deleteRecord = ref(null)

// 分页相关
const currentPage = ref(1)
const pageSize = ref(10)
const pageSizeOptions = [5, 10, 20, 50]

// 表单数据
const teams = ['裁切', '磨边', '平钢化', '弯钢化', '中空', '搬运', '干夹']
const deductTypes = ['破损扣款', '帮忙扣款', '质量扣款', '迟到扣款', '其他扣款']

const formData = ref({
  id: 0,
  team: '',
  deduct_type: '',
  amount: '',
  reason: ''  // 默认为空，提交时会自动设置为"无"（如果为空）
})

// 编辑状态
const isEditing = computed(() => {
  return formData.value.id > 0
})

// 计算属性
const isFormValid = computed(() => {
  const { team, deduct_type, amount } = formData.value
  const amountNum = parseFloat(amount)
  return team && deduct_type && amount && !isNaN(amountNum) && amountNum > 0
})

// 筛选选项
const filterOptions = ['全部', ...deductTypes]

// 计算总页数
const totalPages = computed(() => {
  // 确保 filteredList 是数组
  if (!Array.isArray(filteredList.value)) {
    return 1
  }
  return Math.ceil(filteredList.value.length / pageSize.value)
})

// 计算当前页数据
const paginatedList = computed(() => {
  // 确保 filteredList 是数组
  if (!Array.isArray(filteredList.value)) {
    return []
  }
  
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return filteredList.value.slice(start, end)
})

// 加载历史记录
function loadHistory() {
  uni.request({
    url: `${API_BASE}?action=getDeductions`,
    method: 'GET',
    success: res => {
      if (res.data && res.data.ok) {
        historyList.value = res.data.data || []
        filterHistory()
      } else {
        historyList.value = []
        uni.showToast({ 
          title: res.data?.msg || '加载失败', 
          icon: 'none' 
        })
      }
    },
    fail: () => {
      uni.showToast({ 
        title: '网络错误', 
        icon: 'none' 
      })
    }
  })
}

// 筛选历史记录
function filterHistory() {
  // 确保 historyList 是数组
  if (!Array.isArray(historyList.value)) {
    historyList.value = []
  }
  
  const keyword = searchKeyword.value.toLowerCase()
  const filter = filterBy.value
  
  let filtered = [...historyList.value] // 创建副本
  
  // 关键词搜索
  if (keyword) {
    filtered = filtered.filter(item => 
      (item.team && item.team.toLowerCase().includes(keyword)) ||
      (item.deduct_type && item.deduct_type.toLowerCase().includes(keyword))
    )
  }
  
  // 类型筛选
  if (filter && filter !== '全部') {
    filtered = filtered.filter(item => item.deduct_type === filter)
  }
  
  // 确保 filtered 是数组
  if (!Array.isArray(filtered)) {
    filtered = []
  }
  
  filteredList.value = filtered
  // 重置到第一页
  currentPage.value = 1
}

// 跳转到指定页
function goToPage(page) {
  if (page < 1 || page > totalPages.value) return
  currentPage.value = page
}

// 每页条数改变
function onPageSizeChange(e) {
  const newSize = pageSizeOptions[e.detail.value]
  pageSize.value = newSize
  currentPage.value = 1 // 重置到第一页
}

// 提交扣款
async function submitDeduct() {
  if (!isFormValid.value) {
    uni.showToast({ 
      title: '请完善必填信息', 
      icon: 'none' 
    })
    return
  }
  
  const amountNum = parseFloat(formData.value.amount)
  if (isNaN(amountNum) || amountNum <= 0) {
    uni.showToast({ 
      title: '请输入正确的扣款金额', 
      icon: 'none' 
    })
    return
  }
  
  submitting.value = true
  
  try {
    // 如果原因为空，设置为"无"
    const reason = formData.value.reason.trim() || '无'
    
    // 准备请求参数
    let params = {
      action: isEditing.value ? 'updateDeduct' : 'addDeduct',
      team: formData.value.team,
      deduct_type: formData.value.deduct_type,
      amount: amountNum,
      reason: reason
    }
    
    // 如果是编辑模式，添加id参数
    if (isEditing.value) {
      params.id = formData.value.id
    }
    
    const res = await uni.request({
      url: `${API_BASE}?${new URLSearchParams(params).toString()}`,
      method: 'GET'
    })
    
    if (res.data && res.data.ok) {
      uni.showToast({ 
        title: isEditing.value ? '更新成功' : `扣款成功 ¥${amountNum.toFixed(2)}`, 
        icon: 'success' 
      })
      
      // 重置表单
      setTimeout(() => {
        resetForm()
        
        // 如果是编辑模式，刷新历史记录
        if (isEditing.value && currentView.value === 'history') {
          loadHistory()
        }
      }, 1000)
    } else {
      const msg = res.data?.msg || (isEditing.value ? '更新失败' : '提交失败')
      uni.showToast({ 
        title: msg, 
        icon: 'none' 
      })
    }
  } catch (err) {
    console.error('操作失败:', err)
    uni.showToast({ 
      title: '网络错误，请重试', 
      icon: 'none' 
    })
  } finally {
    submitting.value = false
  }
}

// 重置表单
function resetForm() {
  formData.value = {
    id: 0,
    team: '',
    deduct_type: '',
    amount: '',
    reason: ''
  }
}

// 编辑扣款记录
function editDeduct(item) {
  formData.value = {
    id: item.id,
    team: item.team,
    deduct_type: item.deduct_type,
    amount: item.amount.toString(),
    reason: item.reason || ''
  }
  currentView.value = 'add'
}

// 确认删除扣款记录
function confirmDeleteDeduct(item) {
  deleteRecord.value = item
  showDeleteConfirm.value = true
}

// 删除扣款记录
function deleteDeduct() {
  if (!deleteRecord.value) return
  
  uni.request({
    url: `${API_BASE}?action=deleteDeduct&id=${deleteRecord.value.id}`,
    method: 'GET',
    success: res => {
      if (res.data && res.data.ok) {
        uni.showToast({ 
          title: '删除成功', 
          icon: 'success' 
        })
        
        // 刷新历史记录
        loadHistory()
      } else {
        uni.showToast({ 
          title: res.data?.msg || '删除失败', 
          icon: 'none' 
        })
      }
    },
    fail: () => {
      uni.showToast({ 
        title: '网络错误', 
        icon: 'none' 
      })
    },
    complete: () => {
      showDeleteConfirm.value = false
      deleteRecord.value = null
    }
  })
}

// 返回主页
function goBackToMain() {
  uni.navigateTo({
    url: '/pages/index/index'
  })
}

// 格式化日期
function formatDate(dateString) {
  if (!dateString) return ''
  const date = new Date(dateString)
  return date.toLocaleDateString('zh-CN')
}
</script>

<style>
.layout {
  display: flex;
  min-height: 100vh;
  background: #f5f7fb;
}

.sidebar {
  width: 220rpx;
  background: #fff;
  border-right: 1px solid #eee;
  padding: 20rpx;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  position: sticky;
  top: 0;
  height: 100vh;
  overflow-y: auto;
}

.user-card {
  text-align: center;
  margin-bottom: 30rpx;
}

.avatar {
  font-size: 44rpx;
}

.user-box .name {
  font-weight: 700;
}

.logout {
  font-size: 22rpx;
  color: #64748b;
  margin-top: 10rpx;
}

.logout:active {
  color: #ef4444;
}

.menu {
  margin-top: 0;
  flex: 1;
}

.menu-item {
  padding: 12rpx;
  border-radius: 8rpx;
  margin-bottom: 8rpx;
}

.menu-item.active {
  background: #e6f0ff;
  color: #2d8cf0;
}

.footer {
  margin-top: auto;
  text-align: center;
  color: #999;
  font-size: 22rpx;
  padding: 20rpx 0 10rpx;
  border-top: 1px dashed #eee;
}

.main {
  flex: 1;
  padding: 30rpx;
  overflow-y: auto;
}

.card {
  background: #fff;
  padding: 24rpx;
  border-radius: 14rpx;
  margin-bottom: 20rpx;
}

.card-title {
  font-weight: 700;
  margin-bottom: 12rpx;
  display: block;
}

.section-title {
  display: block;
  font-size: 28rpx;
  font-weight: 600;
  color: #1e293b;
  margin-bottom: 20rpx;
  border-left: 6rpx solid #2563eb;
  padding-left: 16rpx;
}

.team-selector {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 16rpx;
}

.team-item {
  padding: 20rpx 0;
  text-align: center;
  background: #f8fafc;
  border: 2rpx solid #e2e8f0;
  border-radius: 10rpx;
  font-size: 26rpx;
  color: #475569;
}

.team-item.active {
  background: #2563eb;
  color: white;
  border-color: #2563eb;
}

.type-selector {
  display: flex;
  flex-wrap: wrap;
  gap: 16rpx;
}

.type-item {
  padding: 16rpx 24rpx;
  background: #f8fafc;
  border: 2rpx solid #e2e8f0;
  border-radius: 10rpx;
  font-size: 24rpx;
  color: #475569;
  flex: 1;
  min-width: 160rpx;
  text-align: center;
}

.type-item.active {
  background: #10b981;
  color: white;
  border-color: #10b981;
}

.amount-input-wrapper {
  display: flex;
  align-items: center;
  background: #f8f8f8;
  border-radius: 8rpx;
  padding: 0 24rpx;
}

.currency {
  font-size: 40rpx;
  font-weight: 600;
  color: #2563eb;
  margin-right: 16rpx;
}

.amount-input {
  flex: 1;
  font-size: 36rpx;
  font-weight: 600;
  color: #1e293b;
  background: transparent;
  border: none;
  outline: none;
  padding: 20rpx 0;
}

.reason-textarea {
  width: 100%;
  height: 160rpx;
  background: #f8f8f8;
  border-radius: 8rpx;
  padding: 20rpx;
  font-size: 26rpx;
  color: #1e293b;
  margin-bottom: 12rpx;
  box-sizing: border-box;
}

.char-count {
  display: block;
  text-align: right;
  font-size: 22rpx;
  color: #64748b;
}

.submit-section {
  background: #fff;
  border-radius: 14rpx;
  padding: 24rpx;
  margin-top: 20rpx;
}

.summary-info {
  background: #f0f9ff;
  border-radius: 10rpx;
  padding: 20rpx;
  margin-bottom: 24rpx;
  border-left: 6rpx solid #2563eb;
}

.summary-info text {
  display: block;
  font-size: 26rpx;
  color: #1e293b;
  margin-bottom: 10rpx;
}

.btn-primary {
  background: #2d8cf0;
  color: #fff;
  padding: 12rpx;
  border-radius: 10rpx;
  margin-top: 16rpx;
}

.btn-primary:disabled {
  background: #cbd5e1;
  opacity: 0.7;
}

.btn-cancel {
  flex: 1;
  background: #f0f0f0;
  color: #666;
  padding: 12rpx;
  border-radius: 10rpx;
  margin-top: 16rpx;
}

.btn-cancel:active {
  background: #e0e0e0;
}

/* 历史记录样式 */
.filter-bar {
  display: flex;
  gap: 10rpx;
  margin-bottom: 20rpx;
}

.search-input {
  flex: 1;
  background: #f8f8f8;
  padding: 12rpx;
  border-radius: 8rpx;
  border: 1px solid #eee;
}

.filter-picker {
  background: #f8f8f8;
  padding: 12rpx 20rpx;
  border-radius: 8rpx;
  border: 1px solid #eee;
  color: #666;
}

.empty-tip {
  text-align: center;
  padding: 40rpx;
  color: #999;
}

.deduct-item {
  background: #f8f9fa;
  border-radius: 12rpx;
  padding: 20rpx;
  margin-bottom: 15rpx;
  border-left: 6rpx solid #2d8cf0;
}

.deduct-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15rpx;
  padding-bottom: 10rpx;
  border-bottom: 1px dashed #ddd;
}

.deduct-date {
  font-weight: 600;
  color: #333;
  font-size: 28rpx;
}

.deduct-amount {
  font-weight: 700;
  color: #fa5151;
  font-size: 32rpx;
}

.deduct-details {
  margin-bottom: 15rpx;
}

.detail-row {
  display: flex;
  margin-bottom: 8rpx;
  font-size: 26rpx;
}

.detail-label {
  color: #666;
  min-width: 140rpx;
}

.detail-value {
  color: #333;
  flex: 1;
}

.deduct-actions {
  display: flex;
  justify-content: flex-end;
  gap: 15rpx;
  padding-top: 15rpx;
  border-top: 1px dashed #ddd;
}

.action-btn {
  padding: 8rpx 20rpx;
  border-radius: 6rpx;
  font-size: 24rpx;
}

.edit-btn {
  background: #e6f0ff;
  color: #2d8cf0;
}

.edit-btn:active {
  background: #d0e0ff;
}

.delete-btn {
  background: #ffe6e6;
  color: #fa5151;
}

.delete-btn:active {
  background: #ffd0d0;
}

/* 分页样式 */
.pagination {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: 30rpx;
  padding-top: 20rpx;
  border-top: 1px solid #eee;
}

.pagination-btn {
  padding: 12rpx 24rpx;
  background: #2d8cf0;
  color: white;
  border-radius: 6rpx;
  font-size: 26rpx;
}

.pagination-btn.disabled {
  background: #ccc;
  color: #999;
  pointer-events: none;
}

.pagination-btn:active:not(.disabled) {
  background: #1a7be0;
}

.pagination-info {
  font-size: 26rpx;
  color: #666;
  text-align: center;
}

.pagination-total {
  color: #999;
  font-size: 22rpx;
  margin-left: 10rpx;
}

/* 每页显示条数选择器 */
.page-size-selector {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  margin-top: 20rpx;
  padding-top: 15rpx;
  border-top: 1px dashed #eee;
}

.page-size-label {
  color: #666;
  font-size: 26rpx;
  margin-right: 10rpx;
}

.page-size-picker {
  background: #f8f8f8;
  padding: 8rpx 16rpx;
  border-radius: 6rpx;
  border: 1px solid #ddd;
  color: #2d8cf0;
  font-size: 26rpx;
}

/* 删除确认弹窗 */
.mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, .4);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.dialog {
  background: #fff;
  padding: 30rpx;
  border-radius: 12rpx;
  width: 80%;
  max-width: 600rpx;
}

.delete-confirm-text {
  display: block;
  margin: 20rpx 0;
  font-size: 28rpx;
  text-align: center;
}

.delete-record-info {
  display: block;
  margin: 10rpx 0;
  padding: 10rpx 0;
  border-bottom: 1px dashed #eee;
}

.dialog-buttons {
  display: flex;
  gap: 20rpx;
  margin-top: 30rpx;
}

.btn-cancel {
  flex: 1;
  background: #f0f0f0;
  color: #666;
  padding: 12rpx;
  border-radius: 10rpx;
}

.btn-cancel:active {
  background: #e0e0e0;
}
</style>