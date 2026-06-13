<template>
  <div>
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:24px;">
      <div>
        <h1 style="color:white;margin-bottom:8px;">数据备份舱</h1>
        <p style="color:rgba(255,255,255,0.8);">手动备份与恢复平台数据</p>
      </div>
      <button class="btn btn-primary" style="padding:12px 28px;font-size:16px;" :disabled="backingUp" @click="handleBackup">
        {{ backingUp ? '备份中...' : '立即备份' }}
      </button>
    </div>

    <div v-if="lastBackupResult" class="card" style="margin-bottom:20px;border-left:4px solid #667eea;">
      <p style="color:#155724;font-weight:500;">
        备份成功！文件：{{ lastBackupResult.filename }}
        | 物品数：{{ lastBackupResult.itemsCount }}
        | 交换记录数：{{ lastBackupResult.exchangesCount }}
        | 大小：{{ formatSize(lastBackupResult.size) }}
      </p>
    </div>

    <div v-if="loading" style="text-align:center;padding:60px;color:white;">
      加载中...
    </div>

    <div v-else-if="backups.length === 0" class="empty-state card">
      <h2>暂无备份</h2>
      <p>点击"立即备份"按钮创建第一份数据备份</p>
    </div>

    <div v-else>
      <div class="card" style="overflow-x:auto;">
        <table style="width:100%;border-collapse:collapse;">
          <thead>
            <tr style="border-bottom:2px solid #e0e0e0;text-align:left;">
              <th style="padding:12px 16px;color:#555;font-size:14px;">备份文件</th>
              <th style="padding:12px 16px;color:#555;font-size:14px;">备份时间</th>
              <th style="padding:12px 16px;color:#555;font-size:14px;">物品数</th>
              <th style="padding:12px 16px;color:#555;font-size:14px;">交换记录数</th>
              <th style="padding:12px 16px;color:#555;font-size:14px;">文件大小</th>
              <th style="padding:12px 16px;color:#555;font-size:14px;">操作</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="b in backups" :key="b.filename" style="border-bottom:1px solid #f0f0f0;">
              <td style="padding:12px 16px;font-family:monospace;font-size:13px;">{{ b.filename }}</td>
              <td style="padding:12px 16px;font-size:13px;">{{ formatDate(b.createdAt) }}</td>
              <td style="padding:12px 16px;">
                <span class="badge badge-available">{{ b.itemsCount }}</span>
              </td>
              <td style="padding:12px 16px;">
                <span class="badge badge-exchanged">{{ b.exchangesCount }}</span>
              </td>
              <td style="padding:12px 16px;font-size:13px;">{{ formatSize(b.size) }}</td>
              <td style="padding:12px 16px;">
                <div style="display:flex;gap:8px;">
                  <button class="btn btn-primary" style="padding:6px 14px;font-size:12px;" @click="handleDownload(b.filename)">
                    下载
                  </button>
                  <button class="btn btn-danger" style="padding:6px 14px;font-size:12px;" @click="handleDelete(b.filename)">
                    删除
                  </button>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { createBackup, getBackupList, downloadBackup, deleteBackup } from '../api/index.js'

const backups = ref([])
const loading = ref(true)
const backingUp = ref(false)
const lastBackupResult = ref(null)

function formatSize(bytes) {
  if (bytes < 1024) return bytes + ' B'
  if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(1) + ' KB'
  return (bytes / (1024 * 1024)).toFixed(2) + ' MB'
}

function formatDate(dateStr) {
  const date = new Date(dateStr)
  const y = date.getFullYear()
  const m = String(date.getMonth() + 1).padStart(2, '0')
  const d = String(date.getDate()).padStart(2, '0')
  const h = String(date.getHours()).padStart(2, '0')
  const min = String(date.getMinutes()).padStart(2, '0')
  const s = String(date.getSeconds()).padStart(2, '0')
  return y + '-' + m + '-' + d + ' ' + h + ':' + min + ':' + s
}

async function loadBackups() {
  loading.value = true
  try {
    backups.value = await getBackupList()
  } catch (e) {
    console.error(e)
  } finally {
    loading.value = false
  }
}

async function handleBackup() {
  backingUp.value = true
  try {
    lastBackupResult.value = await createBackup()
    await loadBackups()
  } catch (e) {
    console.error(e)
    alert('备份失败：' + (e.response?.data?.error || e.message))
  } finally {
    backingUp.value = false
  }
}

async function handleDownload(filename) {
  try {
    const blob = await downloadBackup(filename)
    const url = window.URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = filename
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    window.URL.revokeObjectURL(url)
  } catch (e) {
    console.error(e)
    alert('下载失败')
  }
}

async function handleDelete(filename) {
  if (!confirm('确定删除备份 ' + filename + ' 吗？')) return
  try {
    await deleteBackup(filename)
    await loadBackups()
  } catch (e) {
    console.error(e)
    alert('删除失败')
  }
}

onMounted(loadBackups)
</script>
