<template>
  <el-card>
  <div class="subject-detail-page">
    <!-- 骨架屏加载状态 -->
    <div v-if="loading" class="loading-container">
      <el-skeleton class="hero-skeleton" animated>
        <template #template>
          <div style="display: flex; gap: 24px; padding: 20px;">
            <el-skeleton-item variant="image" style="width: 240px; height: 360px; border-radius: 8px" />
            <div style="flex: 1">
              <el-skeleton-item variant="h1" style="width: 50%; margin-bottom: 20px" />
              <el-skeleton-item variant="text" style="margin-bottom: 10px" :rows="3" />
              <el-skeleton-item variant="text" style="width: 30%" />
            </div>
          </div>
        </template>
      </el-skeleton>
    </div>

    <div v-else-if="subject" class="content-container">
      <!-- 1. 顶部沉浸式区域 (Hero Section) -->
      <div class="hero-section">
        <!-- 模糊背景 -->
        <div class="hero-bg" :style="{ backgroundImage: `url(${getCoverImage(subject)})` }"></div>
        <div class="hero-overlay"></div>

        <div class="hero-content">
          <!-- 左侧海报 -->
          <div class="poster-wrapper">
            <el-image 
              :src="getCoverImage(subject)" 
              fit="cover" 
              class="main-poster"
              :preview-src-list="[getCoverImage(subject)]"
            />
          </div>

          <!-- 右侧主要信息 -->
          <div class="info-wrapper">
            <div class="header-actions">
              <el-tag effect="dark" round color="#fb7299" style="border:none">{{ mapType(subject?.type) }}</el-tag>
              <el-button round size="small" @click="toComments" icon="ChatLineSquare">
                查看评论
              </el-button>
            </div>

            <h1 class="title">{{ subject?.name_cn || subject?.name }}</h1>
            <h2 v-if="subject?.name_cn && subject?.name !== subject?.name_cn" class="subtitle">
              {{ subject?.name }}
            </h2>

            <!-- 评分与排名数据块 -->
            <div class="stats-row">
              <div class="stat-item score">
                <span class="label">评分</span>
                <span class="value">{{ subject?.rating?.score ?? 'N/A' }}</span>
                <el-rate
                  v-model="ratingValue"
                  disabled
                  show-score
                  text-color="#ff9900"
                  score-template=""
                  :max="10"
                  class="custom-rate"
                />
              </div>
              <el-divider direction="vertical" class="stat-divider"/>
              <div class="stat-item rank">
                <span class="label">排名</span>
                <span class="value">#{{ subject?.rank ?? '-' }}</span>
              </div>
            </div>

            <!-- 简介 -->
            <div class="summary-box">
              <p class="summary-text">{{ subject?.summary || '暂无简介' }}</p>
            </div>
          </div>
        </div>
      </div>

      <!-- 2. 详情内容区域 (白色背景) -->
      <div class="main-body">
        <el-row :gutter="24">
          <!-- 左侧：主要信息流 -->
          <el-col :span="16" :xs="24">
            
            <!-- 详细参数 -->
            <el-card shadow="hover" class="detail-card info-grid-card">
              <template #header><span class="section-title">放送信息</span></template>
              <el-descriptions :column="2" border size="large">
                <el-descriptions-item label="话数">{{ subject?.eps_count ?? subject?.eps ?? '-' }}</el-descriptions-item>
                <el-descriptions-item label="开播日期">{{ subject?.air_date || '-' }}</el-descriptions-item>
                <el-descriptions-item label="放送日">{{ subject?.air_weekday || '-' }}</el-descriptions-item>
                <el-descriptions-item label="类型">{{ mapType(subject?.type) }}</el-descriptions-item>
              </el-descriptions>
            </el-card>

            <!-- 角色列表 (改造成卡片网格) -->
            <div class="section-block">
              <h3 class="section-header">角色 Character</h3>
              <div v-if="subject?.crt?.length || subject?.characters?.length" class="card-grid">
                <div 
                  v-for="(item, index) in (subject?.crt || subject?.characters)" 
                  :key="index" 
                  class="mini-card"
                >
                  <!-- 假设 API 有 images 字段，如果没有则显示默认图标 -->
                  <el-avatar 
                    :size="50" 
                    :src="item.images?.grid || item.image" 
                    shape="square"
                    class="card-avatar"
                  >
                    {{ item.name.charAt(0) }}
                  </el-avatar>
                  <div class="card-info">
                    <div class="card-name" :title="item.name">{{ item.name }}</div>
                    <div class="card-role">{{ item.role_name }}</div>
                  </div>
                </div>
              </div>
              <el-empty v-else description="暂无角色信息" :image-size="60" />
            </div>

          </el-col>

          <!-- 右侧：侧边栏 (制作人员等) -->
          <el-col :span="8" :xs="24">
            <el-card shadow="hover" class="detail-card staff-card">
              <template #header><span class="section-title">制作人员 Staff</span></template>
              <div v-if="subject?.staff?.length" class="staff-list">
                <div v-for="(staff, index) in subject?.staff" :key="index" class="staff-row">
                  <div class="staff-role">{{ staff.role_name }}</div>
                  <div class="staff-name">{{ staff.name }}</div>
                </div>
              </div>
              <el-empty v-else description="暂无制作信息" :image-size="60" />
            </el-card>
          </el-col>
        </el-row>
      </div>
    </div>
  </div>
  </el-card>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { ChatLineSquare } from '@element-plus/icons-vue' // 需要确保安装了 icons
import api from '../api/client'

const route = useRoute()
const router = useRouter()

const subject = ref<any>(null)
const loading = ref(false)

// 计算属性：用于 el-rate 组件，将 API 的评分转换为数字
const ratingValue = computed(() => {
  return subject.value?.rating?.score ? Number(subject.value.rating.score) : 0
})

const load = async () => {
  loading.value = true
  try {
    const id = route.params.id
    if (!id) return
    const { data } = await api.get(`/subject/${id}`, { params: { responseGroup: 'large' } })
    subject.value = data
  } finally {
    loading.value = false
  }
}

// 辅助函数：安全获取图片
const getCoverImage = (sub: any) => {
  return sub?.images?.large || sub?.images?.common || sub?.images?.grid || ''
}

const toComments = () => router.push(`/subject/${route.params.id}/comments`)

const mapType = (t: number | undefined) => {
  const map: Record<number, string> = { 
    1: '书籍', 2: '动画', 3: '音乐', 4: '游戏', 6: '三次元' 
  }
  return map[t ?? 0] || '未知'
}

onMounted(load)
</script>

<style scoped>
/* 全局容器 */
.subject-detail-page {
  background-color: #f4f5f7;
  min-height: 100vh;
  padding-bottom: 40px;
}

.loading-container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
  background: #fff;
}

/* ================= Hero Section (顶部沉浸区) ================= */
.hero-section {
  position: relative;
  width: 100%;
  height: 420px; /* 固定高度 */
  overflow: hidden;
  color: #fff;
  display: flex;
  justify-content: center;
}

/* 模糊背景 */
.hero-bg {
  position: absolute;
  top: -10%;
  left: -10%;
  width: 120%;
  height: 120%;
  background-size: cover;
  background-position: center;
  filter: blur(20px) brightness(0.6); /* 模糊且变暗 */
  z-index: 1;
}

/* 渐变遮罩 */
.hero-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: linear-gradient(to bottom, rgba(0,0,0,0.2) 0%, rgba(0,0,0,0.8) 100%);
  z-index: 2;
}

/* 内容容器 */
.hero-content {
  position: relative;
  z-index: 3;
  max-width: 1200px;
  width: 100%;
  padding: 30px 20px;
  display: flex;
  align-items: flex-end; /* 底部对齐 */
  gap: 30px;
}

/* 海报 */
.poster-wrapper {
  flex-shrink: 0;
  width: 220px;
  height: 310px;
  border-radius: 8px;
  box-shadow: 0 8px 16px rgba(0,0,0,0.5);
  overflow: hidden;
  border: 3px solid #fff;
  transform: translateY(40px); /* 让海报稍微突出到白色区域 */
  background: #eee;
}
.main-poster {
  width: 100%;
  height: 100%;
}

/* 右侧信息 */
.info-wrapper {
  flex: 1;
  padding-bottom: 10px;
  display: flex;
  flex-direction: column;
}

.header-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.title {
  margin: 0;
  font-size: 28px;
  font-weight: 700;
  text-shadow: 0 2px 4px rgba(0,0,0,0.5);
}
.subtitle {
  margin: 5px 0 15px;
  font-size: 16px;
  color: rgba(255,255,255,0.8);
  font-weight: 400;
}

/* 评分区域 */
.stats-row {
  display: flex;
  align-items: center;
  margin-bottom: 15px;
}
.stat-item {
  display: flex;
  flex-direction: column;
}
.stat-item .label {
  font-size: 12px;
  color: rgba(255,255,255,0.6);
  text-transform: uppercase;
}
.stat-item.score .value {
  font-size: 32px;
  font-weight: bold;
  color: #ffcc00;
  line-height: 1;
}
.stat-item.rank .value {
  font-size: 24px;
  font-weight: bold;
}
.stat-divider {
  height: 30px;
  border-color: rgba(255,255,255,0.3);
  margin: 0 20px;
}
/* 强制覆盖 el-rate 样式 */
.custom-rate :deep(.el-rate__icon) {
  font-size: 14px;
  margin-right: 2px;
}

.summary-box {
  background: rgba(0,0,0,0.3);
  padding: 12px;
  border-radius: 6px;
  backdrop-filter: blur(4px);
}
.summary-text {
  margin: 0;
  font-size: 14px;
  line-height: 1.6;
  color: #eee;
  display: -webkit-box;
  -webkit-line-clamp: 3; /* 最多显示3行 */
  -webkit-box-orient: vertical;
  overflow: hidden;
}

/* ================= Main Body (白色区域) ================= */
.main-body {
  max-width: 1200px;
  margin: 60px auto 0; /* 留出空间给突出的海报 */
  padding: 0 20px;
}

.section-title {
  font-weight: bold;
  border-left: 4px solid #409eff;
  padding-left: 10px;
  font-size: 16px;
}

.section-header {
  margin: 30px 0 16px;
  font-size: 18px;
  color: #333;
  font-weight: 600;
  display: flex;
  align-items: center;
}
.section-header::before {
  content: '';
  display: inline-block;
  width: 6px;
  height: 6px;
  background: #fb7299;
  border-radius: 50%;
  margin-right: 8px;
}

.detail-card {
  margin-bottom: 24px;
  border-radius: 8px;
  border: none;
  box-shadow: 0 2px 12px rgba(0,0,0,0.05);
}

/* 角色卡片网格 */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
  gap: 12px;
}
.mini-card {
  background: #fff;
  border-radius: 6px;
  padding: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
  border: 1px solid #ebeef5;
  transition: all 0.2s;
}
.mini-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  border-color: #d9ecff;
}
.card-info {
  flex: 1;
  overflow: hidden;
}
.card-name {
  font-size: 13px;
  font-weight: bold;
  color: #333;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.card-role {
  font-size: 12px;
  color: #999;
  margin-top: 2px;
}

/* 制作人员列表 */
.staff-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}
.staff-row {
  display: flex;
  justify-content: space-between;
  padding-bottom: 8px;
  border-bottom: 1px dashed #eee;
  font-size: 13px;
}
.staff-row:last-child {
  border-bottom: none;
}
.staff-role {
  color: #909399;
}
.staff-name {
  font-weight: 500;
  color: #303133;
}

/* 移动端适配 */
@media screen and (max-width: 768px) {
  .hero-section {
    height: auto;
    padding-bottom: 20px;
  }
  .hero-content {
    flex-direction: column;
    align-items: center;
    text-align: center;
  }
  .poster-wrapper {
    width: 160px;
    height: 220px;
    transform: translateY(0);
    margin-bottom: 15px;
  }
  .info-wrapper {
    width: 100%;
  }
  .header-actions {
    justify-content: center;
    gap: 10px;
  }
  .stats-row {
    justify-content: center;
  }
  .main-body {
    margin-top: 20px;
  }
  .summary-text {
    -webkit-line-clamp: 5;
  }
}
</style>