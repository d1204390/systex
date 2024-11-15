<template>
  <div class="employee-manage">
    <div class="content-wrapper">
      <h1 class="page-title">員工管理</h1>

      <!-- 功能按鈕 -->
      <div class="tool-bar">
        <el-button type="info" @click="resetFilters">
          <el-icon><Refresh /></el-icon>重置篩選
        </el-button>
        <el-button type="primary" @click="openAddEmployeeDialog">
          <el-icon><Plus /></el-icon>建立使用者
        </el-button>
      </div>

      <!-- 搜尋和篩選 -->
      <div class="filter-section">
        <el-input v-model="searchQuery" placeholder="姓名關鍵字搜尋" class="search-input">
          <template #prefix>
            <el-icon><Search /></el-icon>
          </template>
        </el-input>

        <el-select v-model="departmentFilter" placeholder="部門篩選">
          <el-option label="全部" value="" />
          <el-option label="業務部" value="業務部" />
          <el-option label="消金部" value="消金部" />
          <el-option label="借貸部" value="借貸部" />
        </el-select>

        <el-select v-model="positionFilter" placeholder="職位篩選">
          <el-option label="全部" value="" />
          <el-option label="經理" value="經理" />
          <el-option label="主管" value="主管" />
          <el-option label="科員" value="科員" />
        </el-select>

        <el-select v-model="sortOrder" placeholder="排序方式">
          <el-option label="員編 (升冪)" value="asc" />
          <el-option label="員編 (降冪)" value="desc" />
        </el-select>
      </div>

      <!-- 表格卡片 -->
      <el-card class="table-card">
        <el-table :data="filteredEmployeeData" style="width: 100%">
          <el-table-column prop="id" label="員編" width="100" />
          <el-table-column prop="name" label="姓名" width="120" />
          <el-table-column prop="department" label="部門" width="120" />
          <el-table-column prop="position" label="職位" width="120" />
          <el-table-column prop="extension" label="分機" width="100" />
          <el-table-column prop="email" label="Email" min-width="200" />
          <el-table-column label="操作" fixed="right" width="200">
            <template #default="scope">
              <el-button-group>
                <el-button type="primary" @click="handleView(scope.row)" :icon="View" size="small" />
                <el-button type="primary" @click="handleEdit(scope.row)" :icon="Edit" size="small" />
                <el-button type="danger" @click="handleDelete(scope.row)" :icon="Delete" size="small" />
              </el-button-group>
            </template>
          </el-table-column>
        </el-table>
      </el-card>

      <!-- 新增/編輯對話框 -->
      <el-dialog v-model="dialogVisible" :title="isEdit ? '編輯員工' : '新增員工'" width="50%">
        <el-form
            ref="formRef"
            :model="form"
            :rules="rules"
            label-width="80px"
        >
          <el-form-item label="姓名" prop="name">
            <el-input v-model="form.name" />
          </el-form-item>
          <el-form-item label="部門" prop="department">
            <el-select v-model="form.department" placeholder="請選擇部門" @change="generateDefaultAccount">
              <el-option label="業務部" value="業務部" />
              <el-option label="消金部" value="消金部" />
              <el-option label="借貸部" value="借貸部" />
            </el-select>
          </el-form-item>
          <el-form-item label="職位" prop="position">
            <el-select v-model="form.position" placeholder="請選擇職位" @change="generateDefaultAccount">
              <el-option label="經理" value="經理" />
              <el-option label="主管" value="主管" />
              <el-option label="科員" value="科員" />
            </el-select>
          </el-form-item>
          <el-form-item label="Email" prop="email">
            <el-input
                v-model="form.email"
                type="email"
                placeholder="請輸入電子郵件"
            />
          </el-form-item>
          <el-form-item label="帳號">
            <el-input v-model="form.username" disabled placeholder="系統自動生成" />
          </el-form-item>
          <el-form-item label="密碼">
            <el-input v-model="form.password" disabled placeholder="系統自動生成" />
          </el-form-item>
        </el-form>
        <template #footer>
          <span class="dialog-footer">
            <el-button @click="dialogVisible = false">取消</el-button>
            <el-button type="primary" @click="handleSubmit" :loading="isSubmitting">
              {{ isSubmitting ? '處理中...' : '確定' }}
            </el-button>
          </span>
        </template>
      </el-dialog>

      <!-- 進度對話框 -->
      <el-dialog
          v-model="progressVisible"
          title="處理進度"
          :close-on-click-modal="false"
          :show-close="false"
          width="30%"
      >
        <div class="progress-container">
          <el-steps :active="activeStep" finish-status="success">
            <el-step title="保存資料" />
            <el-step title="生成帳號" />
            <el-step title="發送郵件" />
          </el-steps>

          <div class="progress-message">
            {{ progressMessage }}
          </div>
        </div>
      </el-dialog>
    </div>
  </div>
</template>

<script setup>
import { ref, reactive, onMounted, computed } from 'vue'
import { Plus, Edit, View, Delete, Search, Refresh } from '@element-plus/icons-vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import axios from 'axios'

// ref 宣告
const searchQuery = ref('')
const departmentFilter = ref('')
const positionFilter = ref('')
const sortOrder = ref('')
const dialogVisible = ref(false)
const isEdit = ref(false)
const employeeData = ref([])
const formRef = ref(null)
const isSubmitting = ref(false)

// 進度相關的 ref
const progressVisible = ref(false)
const activeStep = ref(0)
const progressMessage = ref('')

// 部門代碼映射
const departmentMap = {
  'BD': '業務部',
  'FD': '消金部',
  'LD': '借貸部'
}

// 職位代碼映射
const positionMap = {
  'M': '經理',
  'S': '主管',
  'C': '科員'
}

// 表單驗證規則
const rules = {
  name: [
    { required: true, message: '請輸入姓名', trigger: 'blur' }
  ],
  department: [
    { required: true, message: '請選擇部門', trigger: 'change' }
  ],
  position: [
    { required: true, message: '請選擇職位', trigger: 'change' }
  ],
  email: [
    { required: true, message: '請輸入電子郵件', trigger: 'blur' },
    { type: 'email', message: '請輸入有效的電子郵件地址', trigger: 'blur' }
  ]
}

const form = reactive({
  name: '',
  department: '',
  position: '',
  extension: '',
  username: '',
  password: '',
  email: ''
})

// 部門職位對應表
const prefixMap = {
  '業務部': { '經理': 'BDM', '主管': 'BDS', '科員': 'BDC' },
  '消金部': { '經理': 'FDM', '主管': 'FDS', '科員': 'FDC' },
  '借貸部': { '經理': 'LDM', '主管': 'LDS', '科員': 'LDC' }
}

// 轉換部門代碼為中文
const convertDepartment = (code) => {
  return departmentMap[code] || code
}

// 轉換職位代碼為中文
const convertPosition = (code) => {
  return positionMap[code] || code
}

// 獲取員工列表方法
const fetchEmployees = async () => {
  try {
    const response = await axios.get('/api/user/employees')
    employeeData.value = response.data.map(user => ({
      id: user.account,
      name: user.name,
      department: convertDepartment(user.department),
      position: convertPosition(user.position),
      extension: user.extension,
      username: user.account,
      email: user.email,
      raw_department: user.department,
      raw_position: user.position
    }))
  } catch (error) {
    console.error('獲取用戶列表失敗:', error)
    ElMessage.error('獲取用戶列表失敗')
  }
}

// 篩選後的員工數據
const filteredEmployeeData = computed(() => {
  let result = [...employeeData.value]

  if (searchQuery.value) {
    result = result.filter(employee =>
        employee.name.toLowerCase().includes(searchQuery.value.toLowerCase())
    )
  }

  if (departmentFilter.value) {
    result = result.filter(employee =>
        employee.department === departmentFilter.value
    )
  }

  if (positionFilter.value) {
    result = result.filter(employee =>
        employee.position === positionFilter.value
    )
  }

  if (sortOrder.value) {
    result.sort((a, b) => {
      if (sortOrder.value === 'asc') {
        return a.id.localeCompare(b.id)
      } else {
        return b.id.localeCompare(a.id)
      }
    })
  }

  return result
})

// 開啟新增對話框
const openAddEmployeeDialog = () => {
  dialogVisible.value = true
  isEdit.value = false
  resetForm()
  generateDefaultAccount()
}

// 重置表單
const resetForm = () => {
  if (formRef.value) {
    formRef.value.resetFields()
  }
  form.name = ''
  form.department = ''
  form.position = ''
  form.extension = ''
  form.username = ''
  form.password = ''
  form.email = ''
}

// 重置進度
const resetProgress = () => {
  activeStep.value = 0
  progressMessage.value = ''
  progressVisible.value = false
  isSubmitting.value = false
}

// 生成隨機密碼
const generateRandomPassword = () => {
  const characters = 'abcdefghijklmnopqrstuvwxyz0123456789'
  let password = ''
  for (let i = 0; i < 6; i++) {
    const randomIndex = Math.floor(Math.random() * characters.length)
    password += characters[randomIndex]
  }
  return password
}

// 生成預設帳號
const generateDefaultAccount = () => {
  if (!form.department || !form.position) {
    form.username = ''
    form.password = ''
    return
  }

  const prefix = prefixMap[form.department]?.[form.position]
  if (!prefix) {
    form.username = ''
    form.password = ''
    return
  }

  const existingEmployees = employeeData.value.filter(employee =>
      employee.username?.startsWith(prefix)
  )
  const newNumber = String(existingEmployees.length + 1).padStart(3, '0')
  form.username = `${prefix}${newNumber}`
  form.password = generateRandomPassword()
}

// 查看處理
const handleView = () => {
  ElMessage.info('查看功能開發中')
}

// 編輯處理
const handleEdit = (row) => {
  isEdit.value = true
  dialogVisible.value = true
  Object.assign(form, {
    ...row,
    department: row.raw_department || row.department,
    position: row.raw_position || row.position,
    email: row.email || ''
  })
}

// 刪除處理
const handleDelete = async (row) => {
  try {
    await ElMessageBox.confirm(
        `確定要刪除 ${row.name} 的資料嗎？`,
        '警告',
        {
          confirmButtonText: '確定',
          cancelButtonText: '取消',
          type: 'warning',
        }
    )

    await axios.delete('/api/user/employee', {
      data: {
        account: row.username
      }
    })

    ElMessage.success('刪除成功')
    await fetchEmployees()
  } catch (error) {
    if (error !== 'cancel') {
      console.error('刪除失敗:', error)
      ElMessage.error('刪除失敗，請重試')
    }
  }
}

// 提交處理
const handleSubmit = async () => {
  if (!formRef.value) return

  try {
    await formRef.value.validate()
    isSubmitting.value = true

    // 顯示進度對話框
    progressVisible.value = true
    progressMessage.value = '正在保存用戶資料...'
    activeStep.value = 1

    // 發送註冊請求
    const response = await axios.post('/api/user/register', {
      name: form.name,
      department: form.department,
      position: form.position,
      account: form.username,
      password: form.password,
      email: form.email
    })

    // 更新進度
    progressMessage.value = '正在生成帳號...'
    activeStep.value = 2
    await new Promise(resolve => setTimeout(resolve, 500))

    progressMessage.value = '正在發送郵件通知...'
    activeStep.value = 3
    await new Promise(resolve => setTimeout(resolve, 1000))

    if (response.status === 201) {
      progressMessage.value = '處理完成！'
      await fetchEmployees()

      setTimeout(() => {
        resetProgress()
        dialogVisible.value = false
        resetForm()
        ElMessage.success('新增成功，帳號密碼已發送至使用者信箱')
      }, 1000)
    }
  } catch (error) {
    resetProgress()
    if (error?.message?.includes('validation')) {
      return
    }
    console.error('新增失敗:', error)
    ElMessage.error('新增失敗，請重試')
  }
}

// 重置篩選
const resetFilters = () => {
  searchQuery.value = ''
  departmentFilter.value = ''
  positionFilter.value = ''
  sortOrder.value = ''
}

// 組件載入時獲取數據
onMounted(() => {
  fetchEmployees()
})
</script>

<style scoped>
.employee-manage {
  min-height: 100vh;
  background-color: #f0f2f5;
  padding: 20px;
}

.content-wrapper {
  max-width: 1200px;  /* 設置最大寬度 */
  margin: 0 auto;     /* 水平置中 */
  padding: 20px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0,0,0,0.1);
}

.page-title {
  text-align: center;
  color: #303133;
  margin-bottom: 30px;
  font-size: 24px;
}

.tool-bar {
  margin-bottom: 20px;
  display: flex;
  justify-content: flex-end;  /* 按鈕靠右 */
}

.filter-section {
  display: flex;
  gap: 16px;
  margin-bottom: 20px;
  flex-wrap: wrap; /* 允許篩選項自動換行 */
  justify-content: space-between;  /* 篩選器均勻分布 */
}

.search-input {
  width: 300px;
}

.table-card {
  margin-top: 20px;
  border-radius: 8px;
  display: flex;
  justify-content: center;
  overflow-x: auto; /* 讓表格在小螢幕上水平滾動 */
}

/* 進度對話框相關樣式 */
.progress-container {
  padding: 20px;
}

.progress-message {
  text-align: center;
  margin-top: 20px;
  color: #409EFF;
  font-size: 14px;
  min-height: 30px;
}

/* Element Plus 組件樣式調整 */
:deep(.el-table) {
  border-radius: 8px;
}

:deep(.el-table th) {
  background-color: #f5f7fa;
  color: #606266;
  font-weight: bold;
}

:deep(.el-select) {
  width: 200px;  /* 統一下拉選單寬度 */
}

:deep(.el-button-group) {
  display: flex;
  gap: 8px;
}

:deep(.el-steps) {
  margin-bottom: 20px;
}

:deep(.el-step__title) {
  font-size: 14px;
}

/* 響應式設計 */
@media (max-width: 1024px) {
  .content-wrapper {
    padding: 10px;
  }

  .page-title {
    font-size: 20px;
  }

  .filter-section {
    flex-direction: column; /* 垂直排列篩選項 */
    align-items: flex-start;
  }

  .search-input, .el-select {
    width: 100%; /* 在小螢幕上占滿寬度 */
  }
}

@media (max-width: 768px) {
  .page-title {
    font-size: 18px;
    text-align: left;
  }

  .tool-bar {
    justify-content: center; /* 工具欄按鈕在小螢幕上居中 */
  }

  .table-card {
    margin-top: 10px;
  }
}

@media (max-width: 480px) {
  .content-wrapper {
    padding: 5px;
  }

  .page-title {
    font-size: 16px;
  }

  .filter-section {
    gap: 8px;
  }

  .search-input {
    width: 100%;
  }

  :deep(.el-table-column) {
    font-size: 14px;
  }

  .progress-container {
    padding: 10px;
  }
}
</style>