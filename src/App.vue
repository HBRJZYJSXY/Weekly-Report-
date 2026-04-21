<template>
  <div class="app">
    <div class="app-container">
      <!-- 侧边栏 -->
      <aside class="sidebar">
        <div class="sidebar-header">
          <h2>📋 日报工具</h2>
        </div>
        <el-menu :default-active="activeMenu" class="sidebar-menu" router @select="handleMenuSelect">
          <el-menu-item index="project">
            <el-icon :size="20">
              <Setting />
            </el-icon>
            <span>项目管理</span>
          </el-menu-item>
          <el-menu-item index="daily">
            <el-icon :size="20">
              <Calendar />
            </el-icon>
            <span>日报记录</span>
          </el-menu-item>
          <el-menu-item index="weekly">
            <el-icon :size="20">
              <DataAnalysis />
            </el-icon>
            <span>周报生成</span>
          </el-menu-item>
        </el-menu>
      </aside>

      <!-- 主内容区 -->
      <main class="main-content">
        <!-- 项目管理 -->
        <div v-if="activeMenu === 'project'" class="content-section">
          <div class="section-header">
            <h2>🏗️ 项目管理</h2>
          </div>
          <div class="project-input">
            <el-input v-model="newProject" placeholder="输入项目名称" @keyup.enter="addProject" clearable />
            <el-button type="primary" @click="addProject" :icon="Plus">
              添加
            </el-button>
          </div>
          <div class="project-list">
            <el-card v-for="project in projects" :key="project.id" class="project-item" :body-style="{ padding: '10px' }">
              <div class="project-card-content">
                <span class="project-name">{{ project.name }}</span>
                <el-button type="danger" size="small" @click="deleteProject(project.id)" :icon="Delete" />
              </div>
            </el-card>
          </div>
          <div v-if="projects.length === 0" class="empty-state">
            <el-empty description="暂无项目，点击添加项目开始记录" />
          </div>
        </div>

        <!-- 日报记录 -->
        <div v-if="activeMenu === 'daily'" class="content-section">
          <div class="section-header">
            <h2>📅 日报记录</h2>
          </div>
          <div class="report-form">
            <el-form label-position="top" :model="formData" style="width: 100%">
              <el-form-item label="项目" required>
                <el-select v-model="selectedProject" placeholder="请选择项目" style="width: 100%">
                  <el-option v-for="project in projects" :key="project.id" :label="project.name" :value="project.id" />
                </el-select>
              </el-form-item>
              <el-form-item label="日期" required>
                <el-date-picker v-model="currentReport.date" type="date" placeholder="选择日期" style="width: 100%" value-format="YYYY-MM-DD" />
              </el-form-item>
              <el-form-item label="工作内容" required>
                <el-input v-model="currentReport.content" type="textarea" placeholder="请输入今日工作内容" :rows="4" />
              </el-form-item>
              <el-form-item label="工时 (小时)" required>
                <el-input-number v-model="currentReport.hours" :min="0.5" :step="0.5" placeholder="请输入工时" style="width: 100%" />
              </el-form-item>
              <el-form-item>
                <el-button type="primary" @click="saveReport" :disabled="!selectedProject || !currentReport.content.trim() || !currentReport.hours" :icon="UploadFilled">
                  保存日报
                </el-button>
                <el-button type="success" @click="generateDailyReport" :disabled="!reports.some(report => report.date === new Date().toISOString().split('T')[0])" :icon="Document" style="margin-left: 10px">
                  生成日报
                </el-button>
              </el-form-item>
            </el-form>
          </div>

          <div class="report-list">
            <h3 class="section-subheader">历史日报</h3>
            <div v-if="selectedProjectReports.length > 0">
              <el-card v-for="report in selectedProjectReports" :key="report.id" class="report-item" :body-style="{ padding: '15px' }">
                <div class="report-card-content">
                  <div class="report-date">
                    <el-tag size="small" type="info">{{ report.date }}</el-tag>
                    <el-tag size="small" type="primary" style="margin-left: 8px">{{ getProjectName(report.projectId) }}</el-tag>
                  </div>
                  <div class="report-content">{{ report.content }}</div>
                  <div class="report-hours">
                    <el-tag size="small" type="success">工时: {{ report.hours }} 小时</el-tag>
                  </div>
                  <el-button type="danger" size="small" @click="deleteReport(report.id)" :icon="Delete" style="margin-top: 10px">
                    删除
                  </el-button>
                </div>
              </el-card>
            </div>
            <div v-else class="empty-state">
              <el-empty description="暂无今日日报记录" />
            </div>
          </div>
        </div>

        <!-- 周报生成 -->
        <div v-if="activeMenu === 'weekly'" class="content-section">
          <div class="section-header">
            <h2>📊 周报生成</h2>
          </div>
          <div class="weekly-form">
            <el-form label-position="top" :model="weeklyForm" style="width: 100%">
              <el-form-item label="开始日期" required>
                <el-date-picker v-model="weeklyStart" type="date" placeholder="选择开始日期" style="width: 100%" value-format="YYYY-MM-DD" />
              </el-form-item>
              <el-form-item label="结束日期" required>
                <el-date-picker v-model="weeklyEnd" type="date" placeholder="选择结束日期" style="width: 100%" value-format="YYYY-MM-DD" />
              </el-form-item>
              <el-form-item>
                <el-button type="primary" @click="generateWeekly" :disabled="!weeklyStart || !weeklyEnd" :icon="Document">
                  生成周报
                </el-button>
                <el-button type="success" @click="exportWeekly" :disabled="!weeklyContent" :icon="Download" style="margin-left: 10px">
                  导出周报
                </el-button>
              </el-form-item>
            </el-form>
          </div>
          <div v-if="weeklyContent" class="weekly-content">
            <el-card :body-style="{ padding: '20px' }">
              <pre>{{ weeklyContent }}</pre>
            </el-card>
          </div>
        </div>

        <!-- 生成的日报 -->
        <div v-if="generatedReport && activeMenu === 'daily'" class="generated-report">
          <div class="section-header">
            <h2>📄 生成的日报</h2>
          </div>
          <div class="report-content-box">
            <el-card :body-style="{ padding: '20px' }">
              <pre>{{ generatedReport }}</pre>
            </el-card>
          </div>
          <el-button type="primary" @click="exportDailyReport" :icon="Download" style="margin-top: 16px">
            导出日报
          </el-button>
        </div>
      </main>
    </div>
  </div>
</template>

<script>
import { Plus, Delete, UploadFilled, Document, Download, Setting, Calendar, DataAnalysis } from '@element-plus/icons-vue'

export default {
  name: 'App',
  components: {
    Plus,
    Delete,
    UploadFilled,
    Document,
    Download,
    Setting,
    Calendar,
    DataAnalysis
  },
  data() {
    return {
      projects: JSON.parse(localStorage.getItem('projects')) || [],
      reports: JSON.parse(localStorage.getItem('reports')) || [],
      newProject: '',
      selectedProject: null,
      currentReport: {
        date: new Date().toISOString().split('T')[0],
        content: '',
        hours: ''
      },
      formData: {
        project: '',
        date: '',
        content: '',
        hours: ''
      },
      weeklyForm: {
        startDate: '',
        endDate: ''
      },
      weeklyStart: '',
      weeklyEnd: '',
      weeklyContent: '',
      generatedReport: '',
      activeMenu: 'project'
    }
  },
  computed: {
    selectedProjectReports() {
      const today = new Date().toISOString().split('T')[0]
      return this.reports.filter(report => report.date === today)
    },
    selectedProjectName() {
      if (!this.selectedProject) return ''
      const project = this.projects.find(p => p.id === this.selectedProject)
      return project ? project.name : ''
    }
  },
  methods: {
    handleMenuSelect(key) {
      this.activeMenu = key
    },
    addProject() {
      if (this.newProject.trim()) {
        const project = {
          id: Date.now().toString(),
          name: this.newProject.trim()
        }
        this.projects.push(project)
        this.newProject = ''
        this.saveToLocalStorage()
      }
    },
    deleteProject(projectId) {
      this.projects = this.projects.filter(p => p.id !== projectId)
      this.reports = this.reports.filter(r => r.projectId !== projectId)
      if (this.selectedProject === projectId) {
        this.selectedProject = null
      }
      this.saveToLocalStorage()
    },
    saveReport() {
      if (!this.selectedProject || !this.currentReport.content.trim() || !this.currentReport.hours) return

      // 检查是否已存在同日期同项目的日报
      const existingIndex = this.reports.findIndex(r => r.projectId === this.selectedProject && r.date === this.currentReport.date)

      if (existingIndex > -1) {
        // 合并内容和工时
        const existingReport = this.reports[existingIndex]
        // 为新内容添加工时
        const contentWithHours = `${this.currentReport.content.trim()} 工时：${this.currentReport.hours}小时`
        existingReport.content += '\n' + contentWithHours
        existingReport.hours += this.currentReport.hours
      } else {
        // 创建新日报
        // 为内容添加工时
        const contentWithHours = `${this.currentReport.content.trim()} 工时：${this.currentReport.hours}小时`
        const report = {
          id: Date.now().toString(),
          projectId: this.selectedProject,
          date: this.currentReport.date,
          content: contentWithHours,
          hours: this.currentReport.hours
        }
        this.reports.push(report)
      }

      this.currentReport = {
        date: new Date().toISOString().split('T')[0],
        content: '',
        hours: ''
      }
      this.generatedReport = ''
      this.saveToLocalStorage()
    },
    deleteReport(reportId) {
      this.reports = this.reports.filter(r => r.id !== reportId)
      this.saveToLocalStorage()
    },
    generateDailyReport() {
      const today = new Date().toISOString().split('T')[0]
      const todayReports = this.reports.filter(report => report.date === today)

      if (todayReports.length === 0) return

      let reportContent = `日报
日期：${today}\n\n`
      let totalHours = 0

      // 按项目分组
      const projectsReports = {}
      todayReports.forEach(report => {
        if (!projectsReports[report.projectId]) {
          projectsReports[report.projectId] = {
            name: this.getProjectName(report.projectId),
            content: report.content,
            hours: report.hours
          }
        } else {
          projectsReports[report.projectId].content += '\n' + report.content
          projectsReports[report.projectId].hours += report.hours
        }
      })

      // 生成日报内容
      Object.values(projectsReports).forEach(projectReport => {
        reportContent += `项目：${projectReport.name}\n`
        // 为工作内容添加序号
        const contentLines = projectReport.content.split('\n')
        contentLines.forEach((line, index) => {
          reportContent += `${index + 1}. ${line}\n`
        })
        reportContent += `工时：${projectReport.hours} 小时\n\n`
        totalHours += projectReport.hours
      })

      reportContent += `总计工时：${totalHours} 小时`
      this.generatedReport = reportContent
    },
    exportDailyReport() {
      if (!this.generatedReport) return

      const blob = new Blob([this.generatedReport], { type: 'text/plain' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = `日报_${this.currentReport.date}.txt`
      a.click()
      URL.revokeObjectURL(url)
    },
    generateWeekly() {
      if (!this.weeklyStart || !this.weeklyEnd) return

      let content = `周报 (${this.weeklyStart} 至 ${this.weeklyEnd})\n\n`

      this.projects.forEach(project => {
        const projectReports = this.reports.filter(r => r.projectId === project.id && r.date >= this.weeklyStart && r.date <= this.weeklyEnd)

        if (projectReports.length > 0) {
          content += `项目：${project.name}\n`
          projectReports.forEach(report => {
            content += `  ${report.date}：${report.content} (${report.hours}小时)\n`
          })
          content += '\n'
        }
      })

      this.weeklyContent = content
    },
    exportWeekly() {
      if (!this.weeklyContent) return

      const blob = new Blob([this.weeklyContent], { type: 'text/plain' })
      const url = URL.createObjectURL(blob)
      const a = document.createElement('a')
      a.href = url
      a.download = `周报_${this.weeklyStart}_${this.weeklyEnd}.txt`
      a.click()
      URL.revokeObjectURL(url)
    },
    saveToLocalStorage() {
      localStorage.setItem('projects', JSON.stringify(this.projects))
      localStorage.setItem('reports', JSON.stringify(this.reports))
    },
    getProjectName(projectId) {
      const project = this.projects.find(p => p.id === projectId)
      return project ? project.name : '未知项目'
    }
  }
}
</script>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  line-height: 1.6;
  color: #333;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  min-height: 100vh;
}

.app {
  width: 100%;
  min-height: 100vh;
  padding: 20px;
}

.app-container {
  display: flex;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-radius: 12px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  min-height: calc(100vh - 40px);
}

/* 侧边栏 */
.sidebar {
  width: 250px;
  background: linear-gradient(135deg, #4f46e5 0%, #7c3aed 100%);
  color: white;
  display: flex;
  flex-direction: column;
  transition: all 0.3s ease;
}

.sidebar-header {
  padding: 24px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.sidebar-header h2 {
  font-size: 1.2rem;
  font-weight: 600;
  margin: 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.sidebar-menu {
  flex: 1;
  margin-top: 20px;
}

/* 主内容区 */
.main-content {
  flex: 1;
  padding: 30px;
  overflow-y: auto;
  background: rgba(255, 255, 255, 0.9);
}

.content-section {
  background: white;
  border-radius: 12px;
  padding: 24px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  margin-bottom: 24px;
  transition: all 0.3s ease;
}

.content-section:hover {
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.12);
  transform: translateY(-2px);
}

.section-header {
  margin-bottom: 20px;
  padding-bottom: 12px;
  border-bottom: 2px solid #e2e8f0;
}

.section-header h2 {
  font-size: 1.3rem;
  color: #2d3748;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 8px;
}

.section-subheader {
  font-size: 1.1rem;
  color: #4a5568;
  margin-bottom: 15px;
  font-weight: 500;
}

/* 项目管理 */
.project-input {
  display: flex;
  margin-bottom: 20px;
  gap: 10px;
}

.project-input .el-input {
  flex: 1;
}

.project-list {
  max-height: 400px;
  overflow-y: auto;
  padding-right: 8px;
}

.project-card-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.project-name {
  font-weight: 500;
  color: #2d3748;
}

/* 日报记录 */
.report-form {
  margin-bottom: 24px;
}

.report-list {
  margin-top: 24px;
}

.report-card-content {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 10px;
}

.report-date {
  align-self: flex-start;
}

.report-content {
  width: 100%;
  line-height: 1.5;
  color: #4a5568;
}

/* 周报生成 */
.weekly-form {
  margin-bottom: 24px;
}

.weekly-content {
  margin-top: 24px;
}

/* 生成的日报 */
.generated-report {
  margin-top: 24px;
  background: white;
  border-radius: 12px;
  padding: 24px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
}

.report-content-box {
  margin-bottom: 16px;
}

.report-content-box pre {
  white-space: pre-wrap;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  line-height: 1.6;
  color: #2d3748;
  font-size: 0.95rem;
  margin: 0;
}

.empty-state {
  margin-top: 20px;
}

/* 滚动条样式 */
::-webkit-scrollbar {
  width: 6px;
}

::-webkit-scrollbar-track {
  background: #f1f1f1;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb {
  background: #c1c1c1;
  border-radius: 3px;
}

::-webkit-scrollbar-thumb:hover {
  background: #a1a1a1;
}

/* 响应式设计 */
@media (max-width: 1024px) {
  .sidebar {
    width: 200px;
  }

  .main-content {
    padding: 20px;
  }
}

@media (max-width: 768px) {
  .app-container {
    flex-direction: column;
  }

  .sidebar {
    width: 100%;
    height: auto;
  }

  .sidebar-menu {
    display: flex;
    flex-direction: row;
    overflow-x: auto;
    margin-top: 0;
  }

  .sidebar-menu .el-menu-item {
    flex: 1;
    min-width: 100px;
    text-align: center;
  }

  .main-content {
    flex: 1;
    padding: 20px;
  }

  .project-input {
    flex-direction: column;
  }

  .project-input .el-button {
    width: 100%;
  }
}

/* 动画效果 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.slide-up-enter-active,
.slide-up-leave-active {
  transition: all 0.3s ease;
}

.slide-up-enter-from {
  transform: translateY(20px);
  opacity: 0;
}

.slide-up-leave-to {
  transform: translateY(-20px);
  opacity: 0;
}

/* Element Plus 自定义样式 */
:deep(.el-card) {
  border-radius: 8px;
  margin-bottom: 10px;
  transition: all 0.3s ease;
}

:deep(.el-card:hover) {
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

:deep(.el-form-item__label) {
  font-weight: 500;
  color: #4a5568;
}

:deep(.el-button) {
  border-radius: 8px;
}

:deep(.el-input__wrapper) {
  border-radius: 8px;
}

:deep(.el-select .el-input__wrapper) {
  border-radius: 8px;
}

:deep(.el-date-picker__wrapper) {
  border-radius: 8px;
}

:deep(.el-input-number) {
  border-radius: 8px;
}

/* 侧边栏菜单样式 */
:deep(.sidebar-menu .el-menu-item) {
  color: rgba(255, 255, 255, 0.8);
  height: 60px;
  line-height: 60px;
  margin: 0 10px;
  border-radius: 8px;
  margin-bottom: 8px;
  transition: all 0.3s ease;
}

:deep(.sidebar-menu .el-menu-item:hover) {
  background: rgba(255, 255, 255, 0.1);
  color: white;
}

:deep(.sidebar-menu .el-menu-item.is-active) {
  background: rgba(255, 255, 255, 0.2);
  color: white;
  font-weight: 600;
}

:deep(.sidebar-menu .el-menu-item .el-icon) {
  margin-right: 10px;
}

/* 移动端菜单 */
@media (max-width: 768px) {
  :deep(.sidebar-menu .el-menu-item) {
    margin: 10px;
    height: 50px;
    line-height: 50px;
  }

  :deep(.sidebar-menu .el-menu-item .el-icon) {
    margin-right: 5px;
  }
}
</style>