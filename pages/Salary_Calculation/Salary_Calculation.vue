<template>
  <view class="container">
    <!-- 页面标题 -->
    <view class="page-header">
      <text class="page-title">员工出勤统计与平均工资计算</text>
    </view>

    <!-- 筛选条件区域 -->
    <view class="filter-card">
      <text class="section-title">查询条件</text>
      
      <!-- 班组选择 -->
      <view class="filter-row">
        <text class="filter-label">选择班组：</text>
        <picker 
          :range="teams" 
          range-key="name" 
          @change="onTeamChange"
          :value="teamIndex"
        >
          <view class="picker">{{ selectedTeam || '请选择班组' }}</view>
        </picker>
      </view>
      
      <!-- 年月选择 -->
      <view class="filter-row">
        <text class="filter-label">选择年月：</text>
        <picker 
          mode="date" 
          fields="year-month" 
          @change="onDateChange"
          :value="selectedDate"
        >
          <view class="picker">{{ selectedDate || '请选择年月' }}</view>
        </picker>
      </view>
      
      <!-- 时间范围显示 
      <view v-if="timeRange" class="time-range">
        <text class="range-label">统计周期：</text>
        <text class="range-value">{{ timeRange }}</text>
      </view>-->
      
      <!-- 计算按钮 -->
      <button 
        class="calculate-btn" 
        :disabled="!canCalculate" 
        @click="calculateAvgSalary"
        :class="{disabled: !canCalculate}"
      >
        计算平均工资
      </button>
    </view>

    <!-- 计算结果区域 -->
    <view v-if="resultData" class="result-card">
      <text class="section-title">计算结果</text>
      
      <view class="result-grid">
        <view class="result-item">
          <text class="result-label">统计周期</text>
          <text class="result-value">{{ resultData.start_date }} 至 {{ resultData.end_date }}</text>
        </view>
        
        <view class="result-item">
          <text class="result-label">工资总额</text>
          <text class="result-value salary">{{ resultData.total_salary.toFixed(2) }} 元</text>
        </view>
        
        <view class="result-item">
          <text class="result-label">出勤总天数</text>
          <text class="result-value">{{ resultData.total_days.toFixed(1) }} 天</text>
        </view>
        
        <view class="result-item">
          <text class="result-label">员工人数</text>
          <text class="result-value">{{ resultData.employee_count }} 人</text>
        </view>
        
        <view class="result-item highlight">
          <text class="result-label">平均工资</text>
          <text class="result-value avg-salary">{{ resultData.avg_salary.toFixed(2) }} 元/天</text>
        </view>
      </view>
      
      <!-- 人均出勤天数 -->
      <view v-if="resultData.employee_count > 0" class="avg-attendance">
        <text>人均出勤天数：{{ (resultData.total_days / resultData.employee_count).toFixed(1) }} 天</text>
      </view>
    </view>

    <!-- 员工出勤明细 -->
    <view v-if="resultData && resultData.attendance_details.length > 0" class="detail-card">
      <text class="section-title">员工出勤明细</text>
      
      <view class="detail-header">
        <text class="col-name">姓名</text>
        <text class="col-job">工种</text>
        <text class="col-days">出勤天数</text>
      </view>
      
      <view 
        v-for="(item, index) in resultData.attendance_details" 
        :key="index" 
        class="detail-row"
      >
        <text class="col-name">{{ item.username }}</text>
        <text class="col-job">{{ item.job_type || '-' }}</text>
        <text class="col-days">{{ item.days.toFixed(1) }}</text>
      </view>
    </view>

    <!-- 出勤记录管理区域 -->
    <view class="management-card">
      <text class="section-title">出勤记录管理</text>
      
      <!-- 添加出勤记录表单 -->
      <view class="add-form">
        <text class="form-title">添加上班记录</text>
        
        <picker 
          :range="userList" 
          range-key="username" 
          @change="onUserSelect"
          :value="selectedUserIndex"
        >
          <view class="form-picker">员工：{{ selectedUser.username || '请选择员工' }}</view>
        </picker>
        
        <picker 
          :range="teams" 
          range-key="name" 
          @change="onAddTeamChange"
          :value="addTeamIndex"
        >
          <view class="form-picker">班组：{{ addForm.team || '请选择班组' }}</view>
        </picker>
        
        <picker 
          mode="date" 
          @change="onAttendanceDateChange"
          :value="addForm.attendance_date"
        >
          <view class="form-picker">出勤日期：{{ addForm.attendance_date || '请选择日期' }}</view>
        </picker>
        
        <view class="form-row">
          <text class="form-label">出勤天数：</text>
          <input 
            v-model="addForm.work_days" 
            class="form-input" 
            type="number" 
            step="0.5" 
            placeholder="1.0"
          />
        </view>
        
        <view class="form-row">
          <text class="form-label">备注：</text>
          <input 
            v-model="addForm.remark" 
            class="form-input" 
            placeholder="可填写备注信息"
          />
        </view>
        
        <button 
          class="add-btn" 
          :disabled="!canAddAttendance" 
          @click="addAttendance"
          :class="{disabled: !canAddAttendance}"
        >
          添加出勤记录
        </button>
      </view>
      
      <!-- 出勤记录列表 -->
      <view class="records-section">
        <view class="records-header">
          <text class="records-title">出勤记录列表</text>
          <button class="refresh-btn" @click="loadAttendanceRecords">刷新</button>
        </view>
        
        <!-- 列表筛选 -->
        <view class="records-filter">
          <picker 
            :range="teams" 
            range-key="name" 
            @change="onFilterTeamChange"
            :value="filterTeamIndex"
          >
            <view class="filter-picker">{{ filterTeam || '全部班组' }}</view>
          </picker>
          
          <picker 
            mode="date" 
            fields="month" 
            @change="onFilterMonthChange"
            :value="filterMonth"
          >
            <view class="filter-picker">{{ filterMonth || '全部月份' }}</view>
          </picker>
        </view>
        
        <!-- 记录列表 -->
        <view v-if="attendanceRecords.length > 0" class="records-list">
          <view 
            v-for="(record, index) in attendanceRecords" 
            :key="record.id" 
            class="record-item"
          >
            <view class="record-main">
              <text class="record-name">{{ record.username }}</text>
              <text class="record-team">{{ record.team }}</text>
              <text class="record-date">{{ record.attendance_date }}</text>
              <text class="record-days">{{ record.work_days }}天</text>
            </view>
            <view class="record-footer">
              <text class="record-job">{{ record.job_type || '-' }}</text>
              <text v-if="record.remark" class="record-remark">{{ record.remark }}</text>
              <text class="record-delete" @click="confirmDeleteRecord(record)">删除</text>
            </view>
          </view>
        </view>
        
        <view v-else class="empty-records">
          <text>暂无出勤记录</text>
        </view>
      </view>
    </view>
  </view>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'

const API = 'http://127.0.0.1/php/api3.php'
const TOKEN_KEY = '__LOGIN_TOKEN__'

// 登录状态
const loginState = ref(false)
const token = ref('')
const user = ref({})

// 筛选数据
const teams = ref([])
const selectedTeam = ref('')
const teamIndex = ref(-1)
const selectedDate = ref('')
const timeRange = ref('')

// 计算结果
const resultData = ref(null)

// 用户列表
const userList = ref([])
const selectedUser = ref({})
const selectedUserIndex = ref(-1)

// 添加出勤记录表单
const addForm = ref({
  user_id: 0,
  username: '',
  team: '',
  job_type: '',
  attendance_date: '',
  work_days: '1.0',
  remark: ''
})

const addTeamIndex = ref(-1)

// 出勤记录列表
const attendanceRecords = ref([])
const filterTeam = ref('')
const filterTeamIndex = ref(-1)
const filterMonth = ref('')

// 计算按钮是否可用
const canCalculate = computed(() => {
  return selectedTeam.value && selectedDate.value
})

// 添加记录按钮是否可用
const canAddAttendance = computed(() => {
  return (
    addForm.value.user_id > 0 &&
    addForm.value.team &&
    addForm.value.attendance_date &&
    addForm.value.work_days
  )
})

onMounted(() => {
  const t = uni.getStorageSync(TOKEN_KEY)
  if (t) {
    token.value = t
    loginState.value = true
    loadTeams()
    loadUsers()
    loadAttendanceRecords()
  } else {
    uni.showToast({
      title: '请先登录',
      icon: 'none'
    })
    setTimeout(() => {
      uni.navigateBack()
    }, 1500)
  }
})

// 加载班组列表
function loadTeams() {
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'getTeams',
      token: token.value
    },
    success: (res) => {
      if (res.data && res.data.ok) {
        teams.value = res.data.data.map(team => ({ name: team }))
      } else {
        uni.showToast({
          title: res.data?.msg || '加载班组失败',
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

// 加载用户列表
function loadUsers() {
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'getUsersForAttendance',
      token: token.value
    },
    success: (res) => {
      if (res.data && res.data.ok) {
        userList.value = res.data.data
      }
    }
  })
}

// 班组选择变化
function onTeamChange(e) {
  const index = e.detail.value
  teamIndex.value = index
  selectedTeam.value = teams.value[index]?.name || ''
  updateTimeRange()
}

// 日期选择变化
function onDateChange(e) {
  selectedDate.value = e.detail.value
  updateTimeRange()
}

// 更新时间范围显示
function updateTimeRange() {
  if (!selectedDate.value) {
    timeRange.value = ''
    return
  }
  
  const date = new Date(selectedDate.value + '-01')
  const year = date.getFullYear()
  const month = date.getMonth() + 1
  
  // 计算上月25日到本月25日
  let prevYear, prevMonth
  if (month === 1) {
    prevYear = year - 1
    prevMonth = 12
  } else {
    prevYear = year
    prevMonth = month - 1
  }
  
  timeRange.value = `${prevYear}年${prevMonth.toString().padStart(2, '0')}月25日 - ${year}年${month.toString().padStart(2, '0')}月25日`
}

// 计算平均工资
function calculateAvgSalary() {
  if (!canCalculate.value) return
  
  const date = new Date(selectedDate.value + '-01')
  const year = date.getFullYear()
  const month = date.getMonth() + 1
  
  uni.showLoading({
    title: '计算中...'
  })
  
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'calculateAvgSalary',
      token: token.value,
      team: selectedTeam.value,
      year: year,
      month: month
    },
    success: (res) => {
      uni.hideLoading()
      if (res.data && res.data.ok) {
        resultData.value = res.data.data
        uni.showToast({
          title: '计算完成',
          icon: 'success'
        })
      } else {
        uni.showToast({
          title: res.data?.msg || '计算失败',
          icon: 'none'
        })
      }
    },
    fail: () => {
      uni.hideLoading()
      uni.showToast({
        title: '网络错误',
        icon: 'none'
      })
    }
  })
}

// 用户选择
function onUserSelect(e) {
  const index = e.detail.value
  selectedUserIndex.value = index
  const user = userList.value[index]
  if (user) {
    selectedUser.value = user
    addForm.value.user_id = user.id
    addForm.value.username = user.username
    addForm.value.job_type = user.job_type
  }
}

// 添加班组选择
function onAddTeamChange(e) {
  const index = e.detail.value
  addTeamIndex.value = index
  addForm.value.team = teams.value[index]?.name || ''
}

// 出勤日期选择
function onAttendanceDateChange(e) {
  addForm.value.attendance_date = e.detail.value
}

// 添加上班记录
function addAttendance() {
  if (!canAddAttendance.value) return
  
  uni.showLoading({
    title: '提交中...'
  })
  
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'addAttendance',
      token: token.value,
      ...addForm.value,
      work_days: parseFloat(addForm.value.work_days)
    },
    success: (res) => {
      uni.hideLoading()
      if (res.data && res.data.ok) {
        uni.showToast({
          title: '添加成功',
          icon: 'success'
        })
        // 清空表单
        addForm.value = {
          user_id: 0,
          username: '',
          team: '',
          job_type: '',
          attendance_date: '',
          work_days: '1.0',
          remark: ''
        }
        selectedUser.value = {}
        selectedUserIndex.value = -1
        addTeamIndex.value = -1
        
        // 刷新记录列表
        loadAttendanceRecords()
      } else {
        uni.showToast({
          title: res.data?.msg || '添加失败',
          icon: 'none'
        })
      }
    },
    fail: () => {
      uni.hideLoading()
      uni.showToast({
        title: '网络错误',
        icon: 'none'
      })
    }
  })
}

// 加载出勤记录
function loadAttendanceRecords() {
  const params = {
    action: 'getAttendance',
    token: token.value
  }
  
  if (filterTeam.value) {
    params.team = filterTeam.value
  }
  
  if (filterMonth.value) {
    const date = new Date(filterMonth.value + '-01')
    const year = date.getFullYear()
    const month = date.getMonth() + 1
    params.start_date = `${year}-${month.toString().padStart(2, '0')}-01`
    
    if (month === 12) {
      params.end_date = `${year + 1}-01-01`
    } else {
      params.end_date = `${year}-${(month + 1).toString().padStart(2, '0')}-01`
    }
  }
  
  uni.request({
    url: API,
    method: 'POST',
    data: params,
    success: (res) => {
      if (res.data && res.data.ok) {
        attendanceRecords.value = res.data.data
      }
    }
  })
}

// 筛选班组变化
function onFilterTeamChange(e) {
  const index = e.detail.value
  filterTeamIndex.value = index
  filterTeam.value = teams.value[index]?.name || ''
  loadAttendanceRecords()
}

// 筛选月份变化
function onFilterMonthChange(e) {
  filterMonth.value = e.detail.value
  loadAttendanceRecords()
}

// 确认删除记录
function confirmDeleteRecord(record) {
  uni.showModal({
    title: '确认删除',
    content: `确定要删除 ${record.username} 在 ${record.attendance_date} 的出勤记录吗？`,
    success: (res) => {
      if (res.confirm) {
        deleteAttendanceRecord(record.id)
      }
    }
  })
}

// 删除出勤记录
function deleteAttendanceRecord(id) {
  uni.showLoading({
    title: '删除中...'
  })
  
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'deleteAttendance',
      token: token.value,
      id: id
    },
    success: (res) => {
      uni.hideLoading()
      if (res.data && res.data.ok) {
        uni.showToast({
          title: '删除成功',
          icon: 'success'
        })
        loadAttendanceRecords()
      } else {
        uni.showToast({
          title: res.data?.msg || '删除失败',
          icon: 'none'
        })
      }
    },
    fail: () => {
      uni.hideLoading()
      uni.showToast({
        title: '网络错误',
        icon: 'none'
      })
    }
  })
}
</script>

<style scoped>
.container {
  padding: 20rpx;
  background: #f5f7fb;
  min-height: 100vh;
}

.page-header {
  margin-bottom: 30rpx;
}

.page-title {
  font-size: 36rpx;
  font-weight: 700;
  color: #333;
  display: block;
  text-align: center;
  padding: 20rpx 0;
}

/* 卡片样式 */
.filter-card,
.result-card,
.detail-card,
.management-card {
  background: white;
  border-radius: 16rpx;
  padding: 30rpx;
  margin-bottom: 30rpx;
  box-shadow: 0 2rpx 12rpx rgba(0, 0, 0, 0.05);
}

.section-title {
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
  margin-bottom: 24rpx;
  display: block;
  border-left: 8rpx solid #2d8cf0;
  padding-left: 16rpx;
}

/* 筛选条件 */
.filter-row {
  display: flex;
  align-items: center;
  margin-bottom: 20rpx;
}

.filter-label {
  width: 140rpx;
  color: #666;
  font-size: 28rpx;
}

.picker {
  flex: 1;
  background: #f8f9fa;
  padding: 20rpx;
  border-radius: 10rpx;
  border: 1px solid #e9ecef;
  color: #333;
  font-size: 28rpx;
}

.time-range {
  background: #e6f0ff;
  padding: 20rpx;
  border-radius: 10rpx;
  margin: 20rpx 0;
}

.range-label {
  color: #2d8cf0;
  font-weight: 600;
  margin-right: 10rpx;
}

.range-value {
  color: #333;
}

/* 按钮样式 */
.calculate-btn,
.add-btn,
.refresh-btn {
  background: #2d8cf0;
  color: white;
  padding: 20rpx;
  border-radius: 10rpx;
  font-size: 28rpx;
  margin-top: 20rpx;
  border: none;
}

.calculate-btn.disabled,
.add-btn.disabled {
  background: #ccc;
  color: #999;
  pointer-events: none;
}

.refresh-btn {
  background: #28a745;
  padding: 10rpx 20rpx;
  font-size: 24rpx;
  margin-top: 0;
}

/* 计算结果 */
.result-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20rpx;
  margin-bottom: 20rpx;
}

.result-item {
  background: #f8f9fa;
  padding: 20rpx;
  border-radius: 10rpx;
  border: 1px solid #e9ecef;
}

.result-label {
  display: block;
  color: #666;
  font-size: 24rpx;
  margin-bottom: 10rpx;
}

.result-value {
  display: block;
  font-size: 32rpx;
  font-weight: 600;
  color: #333;
}

.result-value.salary {
  color: #fa5151;
}

.result-value.avg-salary {
  color: #28a745;
}

.result-item.highlight {
  background: #e6f0ff;
  border-color: #2d8cf0;
  grid-column: span 2;
}

.avg-attendance {
  text-align: center;
  color: #666;
  font-size: 26rpx;
  padding: 10rpx;
  border-top: 1px dashed #ddd;
  margin-top: 10rpx;
}

/* 明细表格 */
.detail-header {
  display: flex;
  background: #f8f9fa;
  padding: 16rpx 20rpx;
  border-radius: 8rpx;
  margin-bottom: 10rpx;
  font-weight: 600;
  color: #333;
}

.detail-row {
  display: flex;
  padding: 16rpx 20rpx;
  border-bottom: 1px solid #f0f0f0;
  align-items: center;
}

.col-name {
  flex: 2;
  color: #333;
}

.col-job {
  flex: 1.5;
  color: #666;
  text-align: center;
}

.col-days {
  flex: 1;
  color: #2d8cf0;
  text-align: right;
  font-weight: 600;
}

/* 管理区域 */
.add-form {
  background: #f8f9fa;
  padding: 24rpx;
  border-radius: 12rpx;
  margin-bottom: 30rpx;
}

.form-title {
  font-size: 28rpx;
  font-weight: 600;
  color: #333;
  margin-bottom: 20rpx;
  display: block;
}

.form-picker {
  background: white;
  padding: 20rpx;
  border-radius: 10rpx;
  border: 1px solid #ddd;
  margin-bottom: 16rpx;
  color: #333;
}

.form-row {
  display: flex;
  align-items: center;
  margin-bottom: 16rpx;
}

.form-label {
  width: 120rpx;
  color: #666;
  font-size: 28rpx;
}

.form-input {
  flex: 1;
  background: white;
  padding: 20rpx;
  border-radius: 10rpx;
  border: 1px solid #ddd;
  color: #333;
}

/* 记录列表 */
.records-section {
  margin-top: 30rpx;
}

.records-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20rpx;
}

.records-title {
  font-size: 28rpx;
  font-weight: 600;
  color: #333;
}

.records-filter {
  display: flex;
  gap: 20rpx;
  margin-bottom: 20rpx;
}

.filter-picker {
  flex: 1;
  background: #f8f9fa;
  padding: 16rpx;
  border-radius: 8rpx;
  border: 1px solid #ddd;
  text-align: center;
  color: #666;
}

.records-list {
  max-height: 400rpx;
  overflow-y: auto;
}

.record-item {
  background: #f8f9fa;
  padding: 20rpx;
  border-radius: 10rpx;
  margin-bottom: 16rpx;
  border: 1px solid #e9ecef;
}

.record-main {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10rpx;
}

.record-name {
  font-weight: 600;
  color: #333;
  flex: 1;
}

.record-team {
  color: #2d8cf0;
  background: #e6f0ff;
  padding: 4rpx 12rpx;
  border-radius: 20rpx;
  font-size: 22rpx;
  margin: 0 10rpx;
}

.record-date {
  color: #666;
  margin: 0 10rpx;
}

.record-days {
  color: #28a745;
  font-weight: 600;
}

.record-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 24rpx;
  color: #999;
}

.record-job {
  color: #666;
}

.record-remark {
  color: #ff9900;
  flex: 1;
  margin-left: 20rpx;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.record-delete {
  color: #fa5151;
  padding: 4rpx 12rpx;
  border-radius: 4rpx;
  border: 1px solid #fa5151;
}

.empty-records {
  text-align: center;
  padding: 40rpx;
  color: #999;
  font-size: 28rpx;
}
</style>
