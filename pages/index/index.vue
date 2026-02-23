<template>
  <view class="layout">

    <!-- 左侧功能栏 -->
    <view class="sidebar">

      <!-- 用户卡片 -->
      <view class="user-card">
        <view v-if="!loginState" class="login-box" @click="showLogin=true">
          <view class="avatar">🔐</view>
          <text class="login-text">点击登录</text>
        </view>

        <view v-else class="user-box">
          <view class="avatar">👤</view>
          <view class="info">
            <text class="name">{{ user.username }}</text>
            <text class="role">{{ user.role }}</text>
          </view>

          <!-- 高级：弱化退出 -->
          <view class="logout" @click="confirmLogout">
            安全退出
          </view>
        </view>
      </view>

      <!-- 菜单 -->
      <view class="menu">
        <view class="menu-item" :class="{active:currentView==='add'}" @click="openView('add')">
          ➕ 工资录入
        </view>
        <view class="menu-item" :class="{active:currentView==='list'}" @click="openView('list')">
          📄 工资记录
        </view>

        <!-- 新增：添加扣款 -->
        <view class="menu-item" @click="goDeduct">
          ➖ 添加扣款
        </view>

        <!-- 新增：工资计算 -->
        <view class="menu-item" @click="goSalaryCalc">
          🧮 工资计算
        </view>

        <view
          v-if="user.role==='admin'"
          class="menu-item"
          :class="{active:currentView==='manage'}"
          @click="openView('manage')"
        >
          ⚙️ 修改工种
        </view>
        <view
          v-if="user.role==='admin'"
          class="menu-item"
          :class="{active:currentView==='users'}"
          @click="openView('users')"
        >
          👥 用户列表
        </view>
      </view>

      <view class="footer">v0.0.1</view>
    </view>

    <!-- 主区 -->
    <view class="main">

      <view class="header">
        <text class="title">工资管理系统</text>
      </view>

      <view class="content">

        <!-- 工资录入 -->
        <view v-if="currentView==='add'" class="card">
          <text class="card-title">工资录入</text>

          <!-- 班组选择 - 按钮组 -->
          <view class="section-title">选择班组</view>
          <view class="btn-group">
            <view 
              v-for="t in teams" 
              :key="t"
              class="btn" 
              :class="{active: team === t}"
              @click="team = t"
            >
              {{ t }}
            </view>
          </view>
          <text v-if="!team" class="error-tip">请选择班组</text>

          <!-- 品类选择 - 按钮组 -->
          <view class="section-title">选择品类</view>
          <view class="btn-group">
            <view 
              v-for="c in categories" 
              :key="c"
              class="btn" 
              :class="{active: category === c}"
              @click="category = c"
            >
              {{ c }}
            </view>
          </view>
          <text v-if="!category" class="error-tip">请选择品类</text>

          <!-- 计薪方式选择 - 按钮组 -->
          <view class="section-title">计薪方式</view>
          <view class="btn-group">
            <view 
              v-for="p in payTypes" 
              :key="p"
              class="btn" 
              :class="{active: payType === p}"
              @click="payType = p"
            >
              {{ p }}
            </view>
          </view>
          <text v-if="!payType" class="error-tip">请选择计薪方式</text>

          <!-- 动态显示输入项 -->
          <view v-if="payType==='按面积'" class="input-section">
            <view class="row">
              <view class="input-label">面积</view>
              <input 
                v-model="area" 
                class="input" 
                :class="{error:!area}"
                placeholder="面积 ㎡" 
                type="number"
              />
            </view>
            <view class="row">
              <view class="input-label">厚度</view>
              <input 
                v-model="thickness" 
                class="input" 
                :class="{error:!thickness}"
                placeholder="厚度 mm" 
                type="number"
              />
            </view>
            <text v-if="payType==='按面积' && (!area || !thickness)" class="error-tip">请填写完整面积和厚度</text>
          </view>

          <view v-if="payType==='按计时'" class="input-section">
            <view class="row">
              <view class="input-label">工时</view>
              <input 
                v-model="hours" 
                class="input" 
                :class="{error:!hours}"
                placeholder="工时" 
                type="number"
              />
            </view>
            <text v-if="payType==='按计时' && !hours" class="error-tip">请填写工时</text>
          </view>

          <view class="input-section">
            <view class="row">
              <view class="input-label">单价/金额</view>
              <input 
                v-model="price" 
                class="input" 
                :class="{error:!price}"
                placeholder="单价 / 金额" 
                type="number"
              />
            </view>
            <text v-if="!price" class="error-tip">请填写单价</text>
          </view>

          <button class="btn-primary" @click="addSalary">提交工资</button>

          <!-- 缓存记录 -->
          <view v-if="cachedRecords.length > 0" class="cache-section">
            <text class="cache-title">📝 待提交记录（{{ cachedRecords.length }}条）</text>
            <view v-for="(record, index) in cachedRecords" :key="record.id" class="cache-item">
              <view class="cache-content">
                <text class="cache-text">{{ record.team }} - {{ record.category }}</text>
                <text class="cache-text">{{ record.pay_type }}：{{ getRecordDetail(record) }}</text>
                <text class="cache-text">单价：{{ record.price }}元</text>
              </view>
              <view class="cache-actions">
                <text class="cache-action" @click="resubmitRecord(index)">重新提交</text>
                <text class="cache-action delete" @click="deleteCachedRecord(index)">删除</text>
              </view>
            </view>
          </view>
        </view>

        <!-- 工资记录 -->
        <view v-if="currentView==='list'" class="card">
          <text class="card-title">工资记录</text>
          
          <!-- 搜索和筛选区域 -->
          <view class="filter-bar">
            <input 
              v-model="searchKeyword" 
              class="search-input" 
              placeholder="搜索班组、品类..." 
              @input="filterSalaryRecords"
            />
            <picker :range="filterOptions" @change="e=>filterBy=e.detail.value">
              <view class="filter-picker">{{ filterBy || '全部' }}</view>
            </picker>
          </view>

          <view v-if="filteredList.length === 0" class="empty-tip">
            <text>暂无工资记录</text>
          </view>

          <!-- 工资记录列表 -->
          <view v-for="item in paginatedList" :key="item.id" class="salary-item">
            <view class="salary-header">
              <text class="salary-date">{{ item.record_date }}</text>
              <text v-if="item.username" class="salary-user">{{ item.username }}</text>
              <text class="salary-amount">￥{{ item.money }}</text>
            </view>
            
            <view class="salary-details">
              <view class="detail-row">
                <text class="detail-label">工种：</text>
                <text class="detail-value">{{ item.job_type }}</text>
              </view>
              <view class="detail-row">
                <text class="detail-label">班组：</text>
                <text class="detail-value">{{ item.team }}</text>
              </view>
              <view class="detail-row">
                <text class="detail-label">品类：</text>
                <text class="detail-value">{{ item.category }}</text>
              </view>
              <view class="detail-row">
                <text class="detail-label">计薪方式：</text>
                <text class="detail-value">{{ getPayTypeName(item.pay_type) }}</text>
              </view>
              
              <view v-if="item.pay_type === 'area'" class="detail-row">
                <text class="detail-label">面积/厚度：</text>
                <text class="detail-value">{{ item.area }}㎡ / {{ item.thickness }}mm</text>
              </view>
              <view v-else-if="item.pay_type === 'time'" class="detail-row">
                <text class="detail-label">工时：</text>
                <text class="detail-value">{{ item.hours }}小时</text>
              </view>
              
              <view class="detail-row">
                <text class="detail-label">单价：</text>
                <text class="detail-value">{{ item.price }}元</text>
              </view>
              
              <view class="detail-row">
                <text class="detail-label">计算金额：</text>
                <text class="detail-value">{{ calculateItemMoney(item) }}</text>
              </view>
              
              <view class="detail-row">
                <text class="detail-label">记录时间：</text>
                <text class="detail-value">{{ formatDateTime(item.created_at) }}</text>
              </view>
            </view>
            
            <view class="salary-actions">
              <view class="action-btn edit-btn" @click="openEditDialog(item)">
                <text>编辑</text>
              </view>
              <view class="action-btn delete-btn" @click="confirmDeleteSalary(item)">
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

        <!-- 管理员：修改工种 -->
        <view v-if="currentView==='manage'" class="card">
          <text class="card-title">修改用户工种</text>

          <picker :range="userList" range-key="username" @change="onSelectUser">
            <view class="picker">用户：{{ selectedUser.username || '请选择' }}</view>
          </picker>

          <picker :range="jobs" @change="onPickJob">
            <view class="picker">新工种：{{ pickJob || '请选择' }}</view>
          </picker>

          <button class="btn-primary" :disabled="!canSetJob" @click="adminSetJob">
            保存修改
          </button>
        </view>

        <!-- 管理员：用户列表 -->
        <view v-if="currentView==='users'" class="card">
          <text class="card-title">用户列表</text>
          <view v-for="u in userList" :key="u.id" class="user-row">
            <view style="display:flex; justify-content:space-between; padding:12rpx 0; border-bottom:1px dashed #eee;">
              <view>
                <text style="font-weight:700;">{{ u.username }}</text>
                <text style="margin-left:20rpx; color:#64748b;">{{ u.role }}</text>
              </view>
              <text style="color:#2d8cf0;">{{ u.job_type || '未设置' }}</text>
            </view>
          </view>
        </view>

      </view>
    </view>

    <!-- 登录弹窗 -->
    <view v-if="showLogin" class="mask" @click.self="showLogin=false">
      <view class="dialog">
        <text class="card-title">系统登录</text>
        <input v-model="username" class="input" placeholder="用户名" />
        <input v-model="password" class="input" password placeholder="密码" />
        <button class="btn-primary" @click="login">登录</button>
      </view>
    </view>

    <!-- 编辑工资弹窗 -->
    <view v-if="showEditDialog" class="mask" @click.self="showEditDialog=false">
      <view class="dialog edit-dialog">
        <text class="card-title">编辑工资记录</text>
        
        <picker :range="teams" @change="e=>editForm.team=teams[e.detail.value]">
          <view class="picker" :class="{error:!editForm.team}">班组：{{ editForm.team || '请选择' }}</view>
        </picker>

        <picker :range="categories" @change="e=>editForm.category=categories[e.detail.value]">
          <view class="picker" :class="{error:!editForm.category}">品类：{{ editForm.category || '请选择' }}</view>
        </picker>

        <picker :range="payTypes" @change="onEditPayTypeChange">
          <view class="picker" :class="{error:!editForm.pay_type}">计薪方式：{{ getPayTypeName(editForm.pay_type) || '请选择' }}</view>
        </picker>

        <view v-if="editForm.pay_type === 'area'" class="row">
          <input 
            v-model="editForm.area" 
            class="input" 
            :class="{error:editForm.pay_type === 'area' && !editForm.area}"
            placeholder="面积 ㎡" 
            type="number"
          />
          <input 
            v-model="editForm.thickness" 
            class="input" 
            :class="{error:editForm.pay_type === 'area' && !editForm.thickness}"
            placeholder="厚度 mm" 
            type="number"
          />
        </view>

        <input 
          v-if="editForm.pay_type === 'time'" 
          v-model="editForm.hours" 
          class="input" 
          :class="{error:editForm.pay_type === 'time' && !editForm.hours}"
          placeholder="工时" 
          type="number"
        />
        
        <input 
          v-model="editForm.price" 
          class="input" 
          :class="{error:!editForm.price}"
          placeholder="单价 / 金额" 
          type="number"
        />
        
        <view class="dialog-buttons">
          <button class="btn-cancel" @click="showEditDialog=false">取消</button>
          <button class="btn-primary" @click="updateSalary">保存修改</button>
        </view>
      </view>
    </view>

  </view>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue'

const API = 'http://127.0.0.1/php/api.php'
const TOKEN_KEY = '__LOGIN_TOKEN__'
const CACHE_KEY = '__SALARY_CACHE__'

const loginState = ref(false)
const showLogin = ref(false)
const user = ref({})
const token = ref('')

const username = ref('')
const password = ref('')

const jobs = ['裁切','磨边','平钢化','弯钢化','中空','搬运','干夹']
const teams = ['裁切组','磨边组','钢化组','弯钢化组','中空组']
// 修改点：在品类数组中添加"补片玻璃"
const categories = ['普通玻璃','Low-E玻璃','补片玻璃']
const payTypes = ['按面积','按计时','帮忙','其他方式']
const payTypeMap = {
  'area': '按面积',
  'time': '按计时', 
  'help': '帮忙',
  'other': '其他方式'
}

// 表单数据 - 默认选择第一个
const team = ref(teams[0] || '')
const category = ref(categories[0] || '')
const payType = ref('按面积')
const area = ref('')
const thickness = ref('')
const hours = ref('')
const price = ref('')
const list = ref([])
const filteredList = ref([])
const searchKeyword = ref('')
const filterBy = ref('')

// 缓存记录
const cachedRecords = ref([])

const userList = ref([])
const selectedUser = ref({})
const pickJob = ref('')
const canSetJob = ref(false)

const currentView = ref('add')

// 编辑相关
const showEditDialog = ref(false)
const editForm = ref({
  id: 0,
  team: '',
  category: '',
  pay_type: '',
  area: '',
  thickness: '',
  hours: '',
  price: ''
})

// 筛选选项
const filterOptions = ['全部', '按面积', '按计时', '帮忙', '其他方式']

// 分页相关
const currentPage = ref(1)
const pageSize = ref(10)
const pageSizeOptions = [5, 10, 20, 50]

onMounted(() => {
  const t = uni.getStorageSync(TOKEN_KEY)
  if (t) verifyToken(t)
  
  // 加载缓存记录
  loadCachedRecords()
})

// 确保 filterList 是数组
watch(filteredList, (newVal) => {
  if (!Array.isArray(newVal)) {
    filteredList.value = []
  }
})

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

// 加载缓存记录
function loadCachedRecords() {
  const cache = uni.getStorageSync(CACHE_KEY)
  if (cache && Array.isArray(cache)) {
    cachedRecords.value = cache
  } else {
    cachedRecords.value = []
  }
}

// 保存缓存记录
function saveCachedRecords() {
  uni.setStorageSync(CACHE_KEY, cachedRecords.value)
}

// 获取记录详情
function getRecordDetail(record) {
  if (record.pay_type === '按面积') {
    return `${record.area}㎡/${record.thickness}mm`
  } else if (record.pay_type === '按计时') {
    return `${record.hours}小时`
  } else if (record.pay_type === '帮忙') {
    return '帮忙'
  } else {
    return '其他方式'
  }
}

// 获取计薪方式名称
function getPayTypeName(payType) {
  return payTypeMap[payType] || payType
}

// 计算单条记录金额说明
function calculateItemMoney(item) {
  if (item.pay_type === 'area') {
    const realArea = item.area / 5 * item.thickness
    return `${item.area}㎡/${item.thickness}mm = ${realArea.toFixed(2)}㎡ × ${item.price}元 = ${item.money}元`
  } else if (item.pay_type === 'time') {
    return `${item.hours}小时 × ${item.price}元 = ${item.money}元`
  } else if (item.pay_type === 'help') {
    return `${item.price}元`
  } else {
    return `${item.price}元`
  }
}

// 格式化日期时间
function formatDateTime(datetime) {
  if (!datetime) return ''
  const date = new Date(datetime)
  return date.toLocaleString('zh-CN')
}

// 确保 list 是数组
function ensureListIsArray() {
  if (!Array.isArray(list.value)) {
    list.value = []
  }
}

// 筛选工资记录
function filterSalaryRecords() {
  // 确保 list 是数组
  ensureListIsArray()
  
  const keyword = searchKeyword.value.toLowerCase()
  const filter = filterBy.value
  
  let filtered = [...list.value] // 创建副本
  
  // 关键词搜索
  if (keyword) {
    filtered = filtered.filter(item => 
      (item.team && item.team.toLowerCase().includes(keyword)) ||
      (item.category && item.category.toLowerCase().includes(keyword)) ||
      (item.job_type && item.job_type.toLowerCase().includes(keyword)) ||
      (item.username && item.username.toLowerCase().includes(keyword))
    )
  }
  
  // 计薪方式筛选
  if (filter && filter !== '全部') {
    const payTypeFilter = filter === '按面积' ? 'area' : 
                         filter === '按计时' ? 'time' : 
                         filter === '帮忙' ? 'help' : 'other'
    filtered = filtered.filter(item => item.pay_type === payTypeFilter)
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

function openView(v) {
  if (!loginState.value) {
    showLogin.value = true
    return
  }
  currentView.value = v
  if (v === 'list') loadSalary()
  if (v === 'manage') refreshUsers()
  if (v === 'users') refreshUsers()
}

function confirmLogout() {
  uni.showModal({
    title: '确认退出',
    content: '是否安全退出当前账号？',
    success(res) {
      if (res.confirm) logout()
    }
  })
}

function verifyToken(t) {
  uni.request({
    url: API,
    method: 'POST',
    data: { action: 'checkToken', token: t },
    success: res => {
      if (res.data && res.data.ok) {
        token.value = t
        user.value = res.data.user
        loginState.value = true
      } else {
        // 令牌无效，清除存储
        uni.removeStorageSync(TOKEN_KEY)
      }
    },
    fail: () => {
      // 网络请求失败，清除存储
      uni.removeStorageSync(TOKEN_KEY)
    }
  })
}

function login() {
  uni.request({
    url: API,
    method: 'POST',
    data: { action: 'login', username: username.value, password: password.value },
    success: res => {
      if (res.data && res.data.ok) {
        token.value = res.data.token
        user.value = res.data.user
        loginState.value = true
        uni.setStorageSync(TOKEN_KEY, token.value)
        showLogin.value = false
      } else {
        uni.showToast({
          title: res.data?.msg || '登录失败',
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

function logout() {
  loginState.value = false
  user.value = {}
  token.value = ''
  list.value = []
  filteredList.value = []
  currentPage.value = 1
  uni.removeStorageSync(TOKEN_KEY)
}

function refreshUsers() {
  uni.request({
    url: API,
    method: 'POST',
    data: { action: 'users', token: token.value },
    success: res => {
      if (res.data && res.data.ok) {
        userList.value = res.data.data || []
      } else {
        userList.value = []
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

function onSelectUser(e) {
  selectedUser.value = userList.value[e.detail.value]
  canSetJob.value = !!pickJob.value
}

function onPickJob(e) {
  pickJob.value = jobs[e.detail.value]
  canSetJob.value = !!selectedUser.value.id
}

function adminSetJob() {
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'setJob',
      token: token.value,
      user_id: selectedUser.value.id,
      job_type: pickJob.value
    },
    success: (res) => {
      if (res.data && res.data.ok) {
        refreshUsers()
        uni.showToast({
          title: '修改成功',
          icon: 'success'
        })
      } else {
        uni.showToast({
          title: res.data?.msg || '修改失败',
          icon: 'none'
        })
      }
    }
  })
}

// 新增：跳转到添加扣款页面（会先检查登录）
function goDeduct() {
  if (!loginState.value) {
    showLogin.value = true
    return
  }
  uni.navigateTo({
    url: '/pages/Salary_Calculation/Salary_Calculation'
  })
}

// 新增：跳转到工资计算页面（会先检查登录）
function goSalaryCalc() {
  if (!loginState.value) {
    showLogin.value = true
    return
  }
  uni.navigateTo({
    url: '/pages/Salary_Calculation/Salary_Calculation'
  })
}

// 验证表单
function validateForm() {
  let isValid = true
  
  // 基础验证
  if (!team.value) isValid = false
  if (!category.value) isValid = false
  if (!payType.value) isValid = false
  if (!price.value) isValid = false
  
  // 按条件验证
  if (payType.value === '按面积') {
    if (!area.value || !thickness.value) isValid = false
  } else if (payType.value === '按计时') {
    if (!hours.value) isValid = false
  }
  
  return isValid
}

function addSalary() {
  // 验证表单
  if (!validateForm()) {
    uni.showToast({
      title: '请填写完整信息',
      icon: 'none'
    })
    return
  }
  
  // 构建请求数据
  const requestData = {
    action: 'addSalary',
    token: token.value,
    team: team.value,
    category: category.value,
    pay_type: payType.value,
    area: Number(area.value) || 0,
    thickness: Number(thickness.value) || 0,
    hours: Number(hours.value) || 0,
    price: Number(price.value) || 0
  }
  
  uni.request({
    url: API,
    method: 'POST',
    data: requestData,
    success: res => {
      if (res.data && res.data.ok) {
        // 成功提交后添加到缓存记录
        const record = {
          id: Date.now(),
          ...requestData,
          timestamp: new Date().toLocaleString()
        }
        cachedRecords.value.unshift(record)
        saveCachedRecords()
        
        // 清除表单数据（保留班组、品类和计薪方式）
        area.value = ''
        thickness.value = ''
        hours.value = ''
        price.value = ''
        
        uni.showToast({
          title: '提交成功',
          icon: 'success'
        })
        
        // 重新加载工资记录
        loadSalary()
      } else {
        uni.showToast({
          title: res.data?.msg || '提交失败',
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

// 重新提交缓存记录
function resubmitRecord(index) {
  const record = cachedRecords.value[index]
  
  // 填充表单
  team.value = record.team
  category.value = record.category
  payType.value = record.pay_type
  area.value = record.area
  thickness.value = record.thickness
  hours.value = record.hours
  price.value = record.price
  
  // 删除原缓存记录
  deleteCachedRecord(index)
}

// 删除缓存记录
function deleteCachedRecord(index) {
  uni.showModal({
    title: '确认删除',
    content: '确定要删除这条记录吗？',
    success: res => {
      if (res.confirm) {
        cachedRecords.value.splice(index, 1)
        saveCachedRecords()
        uni.showToast({
          title: '已删除',
          icon: 'success'
        })
      }
    }
  })
}

function loadSalary() {
  uni.request({
    url: API,
    method: 'POST',
    data: { action: 'salary', token: token.value },
    success: res => {
      if (res.data && res.data.ok) {
        // 确保数据是数组
        list.value = Array.isArray(res.data.data) ? res.data.data : []
      } else {
        list.value = []
        uni.showToast({
          title: res.data?.msg || '加载失败',
          icon: 'none'
        })
      }
      filterSalaryRecords()
    },
    fail: () => {
      uni.showToast({
        title: '网络错误',
        icon: 'none'
      })
    }
  })
}

// 打开编辑对话框
function openEditDialog(item) {
  editForm.value = {
    id: item.id,
    team: item.team || '',
    category: item.category || '',
    pay_type: item.pay_type || '',
    area: item.area || '',
    thickness: item.thickness || '',
    hours: item.hours || '',
    price: item.price || ''
  }
  showEditDialog.value = true
}

// 编辑计薪方式变化
function onEditPayTypeChange(e) {
  const selected = payTypes[e.detail.value]
  // 转换为数据库格式
  if (selected === '按面积') editForm.value.pay_type = 'area'
  else if (selected === '按计时') editForm.value.pay_type = 'time'
  else if (selected === '帮忙') editForm.value.pay_type = 'help'
  else editForm.value.pay_type = 'other'
}

// 验证编辑表单
function validateEditForm() {
  const form = editForm.value
  let isValid = true
  
  if (!form.team) isValid = false
  if (!form.category) isValid = false
  if (!form.pay_type) isValid = false
  if (!form.price) isValid = false
  
  if (form.pay_type === 'area') {
    if (!form.area || !form.thickness) isValid = false
  } else if (form.pay_type === 'time') {
    if (!form.hours) isValid = false
  }
  
  return isValid
}

// 更新工资记录
function updateSalary() {
  if (!validateEditForm()) {
    uni.showToast({
      title: '请填写完整信息',
      icon: 'none'
    })
    return
  }
  
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'updateSalary',
      token: token.value,
      id: editForm.value.id,
      team: editForm.value.team,
      category: editForm.value.category,
      pay_type: editForm.value.pay_type,
      area: Number(editForm.value.area) || 0,
      thickness: Number(editForm.value.thickness) || 0,
      hours: Number(editForm.value.hours) || 0,
      price: Number(editForm.value.price) || 0
    },
    success: res => {
      if (res.data && res.data.ok) {
        uni.showToast({
          title: '更新成功',
          icon: 'success'
        })
        showEditDialog.value = false
        loadSalary()
      } else {
        uni.showToast({
          title: res.data?.msg || '更新失败',
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

// 确认删除工资记录
function confirmDeleteSalary(item) {
  uni.showModal({
    title: '确认删除',
    content: `确定要删除 ${item.record_date} 的工资记录吗？`,
    success: res => {
      if (res.confirm) {
        deleteSalary(item.id)
      }
    }
  })
}

// 删除工资记录
function deleteSalary(id) {
  uni.request({
    url: API,
    method: 'POST',
    data: {
      action: 'deleteSalary',
      token: token.value,
      id: id
    },
    success: res => {
      if (res.data && res.data.ok) {
        uni.showToast({
          title: '删除成功',
          icon: 'success'
        })
        loadSalary()
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
    }
  })
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
  justify-content: flex-start; /* 改为顶部对齐 */
  position: sticky;
  top: 0;
  height: 100vh;
  overflow-y: auto;
}

.user-card {
  text-align: center;
  margin-bottom: 30rpx; /* 增加底部间距 */
}

.avatar {
  font-size: 44rpx;
}

.login-text {
  color: #2d8cf0;
  margin-top: 10rpx;
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
  margin-top: 0; /* 调整菜单上边距 */
  flex: 1;
}

.menu-item {
  padding: 12rpx;
  border-radius: 8rpx;
  margin-bottom: 8rpx; /* 菜单项之间增加间距 */
}

.menu-item.active {
  background: #e6f0ff;
  color: #2d8cf0;
}

.footer {
  margin-top: auto; /* 将页脚推到底部 */
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
  font-weight: 600;
  color: #333;
  margin: 20rpx 0 10rpx;
  font-size: 28rpx;
}

.btn-group {
  display: flex;
  flex-wrap: wrap;
  gap: 16rpx;
  margin: 10rpx 0;
}

.btn {
  padding: 12rpx 24rpx;
  border-radius: 10rpx;
  background: #f8f8f8;
  font-size: 26rpx;
  border: 1px solid #eee;
  transition: all 0.2s;
}

.btn.active {
  background: #2d8cf0;
  color: white;
  border-color: #2d8cf0;
  transform: scale(1.05);
}

.input,
.picker {
  background: #f8f8f8;
  padding: 12rpx;
  border-radius: 8rpx;
  margin: 10rpx 0;
  border: 1px solid transparent;
}

.input.error,
.picker.error {
  border-color: #fa5151;
  background: #fff5f5;
}

.btn-primary {
  background: #2d8cf0;
  color: #fff;
  padding: 12rpx;
  border-radius: 10rpx;
  margin-top: 20rpx;
}

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

.edit-dialog {
  max-width: 700rpx;
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

/* 缓存记录样式 */
.cache-section {
  margin-top: 30rpx;
  border-top: 1px dashed #eee;
  padding-top: 20rpx;
}

.cache-title {
  font-weight: 600;
  color: #666;
  margin-bottom: 15rpx;
  display: block;
}

.cache-item {
  background: #f8f9fa;
  border-radius: 8rpx;
  padding: 15rpx;
  margin-bottom: 10rpx;
  border-left: 4rpx solid #2d8cf0;
}

.cache-content {
  margin-bottom: 10rpx;
}

.cache-text {
  display: block;
  font-size: 24rpx;
  color: #555;
  margin-bottom: 5rpx;
}

.cache-text:first-child {
  font-weight: 600;
  color: #333;
}

.cache-actions {
  display: flex;
  justify-content: flex-end;
  gap: 20rpx;
  border-top: 1px dashed #ddd;
  padding-top: 10rpx;
}

.cache-action {
  color: #2d8cf0;
  font-size: 22rpx;
}

.cache-action.delete {
  color: #fa5151;
}

.cache-action:active {
  opacity: 0.7;
}

.row {
  display: flex;
  gap: 10rpx;
  align-items: center;
  margin: 10rpx 0;
}

.row .input {
  flex: 1;
}

.input-label {
  width: 120rpx;
  text-align: right;
  padding-right: 16rpx;
  color: #666;
  font-size: 28rpx;
}

.input-section {
  background: #f9f9f9;
  border-radius: 12rpx;
  padding: 16rpx;
  margin: 16rpx 0;
}

.error-tip {
  color: #fa5151;
  font-size: 24rpx;
  margin: 5rpx 0;
  display: block;
}

/* 工资记录样式 */
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

.salary-item {
  background: #f8f9fa;
  border-radius: 12rpx;
  padding: 20rpx;
  margin-bottom: 15rpx;
  border-left: 6rpx solid #2d8cf0;
}

.salary-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 15rpx;
  padding-bottom: 10rpx;
  border-bottom: 1px dashed #ddd;
}

.salary-date {
  font-weight: 600;
  color: #333;
  font-size: 28rpx;
}

.salary-user {
  background: #e6f0ff;
  color: #2d8cf0;
  padding: 4rpx 12rpx;
  border-radius: 20rpx;
  font-size: 22rpx;
}

.salary-amount {
  font-weight: 700;
  color: #fa5151;
  font-size: 32rpx;
}

.salary-details {
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

.salary-actions {
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
</style>
