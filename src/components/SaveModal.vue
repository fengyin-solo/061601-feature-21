<script setup lang="ts">
import { ref, computed } from 'vue'
import { useSaveStore } from '../stores/saveStore'
import type { SaveSlot, CharacterSummary } from '../stores/saveStore'
import { getRarityLabel } from '../utils/gameUtils'

const emit = defineEmits<{
  (e: 'close'): void
}>()

const saveStore = useSaveStore()
const mode = ref<'save' | 'load'>('save')
const expandedSlotId = ref<number | null>(null)

const sortedSaves = computed(() => saveStore.sortedSaves)

function toggleExpand(slotId: number) {
  expandedSlotId.value = expandedSlotId.value === slotId ? null : slotId
}

function saveGame(slotId: number) {
  const slotName = prompt('输入存档名称：', `存档 ${slotId}`)
  if (slotName !== null) {
    saveStore.saveToSlot(slotId, slotName || undefined)
  }
}

function loadGame(slotId: number) {
  if (confirm('确定要加载这个存档吗？当前进度将丢失。')) {
    saveStore.loadFromSlot(slotId)
    emit('close')
  }
}

function deleteSave(slotId: number, event: Event) {
  event.stopPropagation()
  if (confirm('确定要删除这个存档吗？')) {
    saveStore.deleteSlot(slotId)
  }
}

function formatDate(timestamp: number): string {
  const date = new Date(timestamp)
  return date.toLocaleString('zh-CN', {
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit'
  })
}

function getAffinityBarWidth(affinity: number): string {
  return `${Math.max(0, Math.min(100, (affinity / 100) * 100))}%`
}

function getAffinityBarColor(affinity: number): string {
  if (affinity >= 80) return '#ec4899'
  if (affinity >= 60) return '#f472b6'
  if (affinity >= 40) return '#fb923c'
  if (affinity >= 20) return '#fbbf24'
  if (affinity >= 0) return '#94a3b8'
  return '#64748b'
}

function formatRaritySummary(slot: SaveSlot): string {
  const rc = slot.collectionSummary?.rarityCount
  if (!rc) return ''
  const parts: string[] = []
  if (rc.legendary) parts.push(`传说×${rc.legendary}`)
  if (rc.epic) parts.push(`史诗×${rc.epic}`)
  if (rc.rare) parts.push(`稀有×${rc.rare}`)
  if (rc.common) parts.push(`普通×${rc.common}`)
  return parts.join(' ')
}

function createNewSave() {
  const newId = saveStore.getAvailableSlotId()
  saveGame(newId)
}
</script>

<template>
  <Teleport to="body">
    <div class="modal-overlay" @click.self="emit('close')">
      <div class="modal-content save-modal">
        <div class="modal-header">
          <h2>💾 存档管理</h2>
          <button class="close-btn" @click="emit('close')">✕</button>
        </div>

        <div class="mode-tabs">
          <button 
            class="tab-btn" 
            :class="{ active: mode === 'save' }"
            @click="mode = 'save'"
          >
            保存
          </button>
          <button 
            class="tab-btn" 
            :class="{ active: mode === 'load' }"
            @click="mode = 'load'"
          >
            读取
          </button>
        </div>

        <div class="auto-save-row">
          <span class="auto-save-label">自动存档</span>
          <label class="toggle-switch">
            <input 
              type="checkbox" 
              :checked="saveStore.autoSaveEnabled"
              @change="saveStore.toggleAutoSave($event.target.checked)"
            />
            <span class="toggle-slider"></span>
          </label>
        </div>

        <div class="save-list">
          <div
            v-for="save in sortedSaves"
            :key="save.id"
            class="save-item"
            :class="{ expanded: expandedSlotId === save.id }"
          >
            <div class="save-main" @click="toggleExpand(save.id)">
              <div class="save-icon">
                {{ mode === 'save' ? '💾' : '📂' }}
              </div>
              <div class="save-info">
                <span class="save-name">{{ save.name }}</span>
                <span class="save-details">{{ save.time }}</span>
                <div v-if="save.characterSummary?.length" class="save-chips">
                  <span
                    v-for="char in save.characterSummary.filter(c => c.unlocked)"
                    :key="char.id"
                    class="char-chip"
                    :style="{ borderColor: getAffinityBarColor(char.affinity) }"
                  >
                    {{ char.avatar }} {{ char.stage }}
                  </span>
                </div>
              </div>
              <div class="save-meta">
                <span class="save-date">{{ formatDate(save.timestamp) }}</span>
                <div class="save-meta-actions">
                  <span class="expand-hint">{{ expandedSlotId === save.id ? '▲' : '▼' }}</span>
                  <button class="delete-btn" @click="deleteSave(save.id, $event)" title="删除">
                    🗑️
                  </button>
                </div>
              </div>
            </div>

            <div v-if="expandedSlotId === save.id" class="save-detail">
              <div class="detail-section">
                <h4 class="detail-title">👥 角色关系</h4>
                <div class="char-list">
                  <div
                    v-for="char in save.characterSummary"
                    :key="char.id"
                    class="char-row"
                  >
                    <span class="char-avatar">{{ char.avatar }}</span>
                    <span class="char-name">{{ char.name }}</span>
                    <div class="affinity-bar-wrap">
                      <div
                        class="affinity-bar"
                        :style="{ width: getAffinityBarWidth(char.affinity), background: getAffinityBarColor(char.affinity) }"
                      ></div>
                    </div>
                    <span class="char-stage" :style="{ color: getAffinityBarColor(char.affinity) }">{{ char.stage }}</span>
                    <span class="char-affinity">{{ char.affinity }}</span>
                  </div>
                </div>
              </div>

              <div v-if="save.keyEvents?.length" class="detail-section">
                <h4 class="detail-title">📖 关键事件</h4>
                <div class="event-tags">
                  <span v-for="evt in save.keyEvents" :key="evt" class="event-tag">
                    {{ evt }}
                  </span>
                </div>
              </div>

              <div v-if="save.collectionSummary" class="detail-section">
                <h4 class="detail-title">🎴 收藏摘要</h4>
                <div class="collection-info">
                  <span class="collection-total">共 {{ save.collectionSummary.total }} 张</span>
                  <span v-if="formatRaritySummary(save)" class="collection-rarity">{{ formatRaritySummary(save) }}</span>
                  <span v-if="save.collectionSummary.latest" class="collection-latest">最近：{{ save.collectionSummary.latest }}</span>
                </div>
              </div>

              <div class="detail-actions">
                <button v-if="mode === 'load'" class="action-btn load-btn" @click="loadGame(save.id)">
                  📂 加载此存档
                </button>
                <button v-if="mode === 'save'" class="action-btn save-btn" @click="saveGame(save.id)">
                  💾 覆盖保存
                </button>
              </div>
            </div>
          </div>

          <div v-if="sortedSaves.length === 0" class="empty-saves">
            暂无存档
          </div>

          <div v-if="mode === 'save'" class="new-save-btn" @click="createNewSave">
            <span class="plus-icon">+</span>
            <span>创建新存档</span>
          </div>
        </div>
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.save-modal {
  padding: 24px;
  max-width: 560px;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.modal-header h2 {
  font-size: 20px;
  font-weight: 600;
}

.close-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--bg-tertiary);
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-btn:hover {
  background: var(--accent-light);
}

.mode-tabs {
  display: flex;
  gap: 4px;
  background: var(--bg-tertiary);
  padding: 4px;
  border-radius: var(--radius-md);
  margin-bottom: 16px;
}

.tab-btn {
  flex: 1;
  padding: 10px;
  border-radius: var(--radius-sm);
  font-size: 14px;
  font-weight: 500;
  background: transparent;
  color: var(--text-secondary);
}

.tab-btn.active {
  background: var(--bg-secondary);
  color: var(--text-primary);
  box-shadow: 0 2px 8px var(--shadow-color);
}

.auto-save-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
  margin-bottom: 16px;
}

.auto-save-label {
  font-size: 14px;
  font-weight: 500;
}

.toggle-switch {
  position: relative;
  width: 44px;
  height: 24px;
  cursor: pointer;
}

.toggle-switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.toggle-slider {
  position: absolute;
  inset: 0;
  background: #cbd5e1;
  border-radius: 9999px;
  transition: 0.3s;
}

.toggle-slider::before {
  content: '';
  position: absolute;
  height: 18px;
  width: 18px;
  left: 3px;
  bottom: 3px;
  background: white;
  border-radius: 50%;
  transition: 0.3s;
}

.toggle-switch input:checked + .toggle-slider {
  background: var(--accent-primary);
}

.toggle-switch input:checked + .toggle-slider::before {
  transform: translateX(20px);
}

.save-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 480px;
  overflow-y: auto;
  padding-right: 8px;
}

.save-item {
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
  transition: all 0.2s;
  overflow: hidden;
}

.save-item:hover {
  background: var(--accent-light);
}

.save-item.expanded {
  background: var(--bg-tertiary);
  box-shadow: 0 4px 12px var(--shadow-color);
}

.save-main {
  display: flex;
  align-items: center;
  gap: 14px;
  padding: 14px;
  cursor: pointer;
  transition: all 0.2s;
}

.save-main:hover {
  transform: translateX(4px);
}

.save-icon {
  font-size: 28px;
  width: 44px;
  text-align: center;
  flex-shrink: 0;
}

.save-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.save-name {
  font-weight: 600;
  font-size: 15px;
}

.save-details {
  font-size: 12px;
  color: var(--text-secondary);
}

.save-chips {
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
  margin-top: 2px;
}

.char-chip {
  display: inline-flex;
  align-items: center;
  gap: 2px;
  padding: 1px 8px;
  border-radius: 9999px;
  border: 1.5px solid;
  font-size: 11px;
  background: var(--bg-secondary);
  white-space: nowrap;
}

.save-meta {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 6px;
  flex-shrink: 0;
}

.save-date {
  font-size: 11px;
  color: var(--text-muted);
}

.save-meta-actions {
  display: flex;
  align-items: center;
  gap: 6px;
}

.expand-hint {
  font-size: 10px;
  color: var(--text-muted);
}

.delete-btn {
  width: 28px;
  height: 28px;
  border-radius: 6px;
  background: #fee2e2;
  font-size: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.delete-btn:hover {
  background: #fecaca;
}

.save-detail {
  padding: 0 14px 14px;
  border-top: 1px solid var(--bg-secondary);
  margin-top: 0;
  animation: slideDown 0.2s ease;
}

@keyframes slideDown {
  from {
    opacity: 0;
    max-height: 0;
  }
  to {
    opacity: 1;
    max-height: 500px;
  }
}

.detail-section {
  margin-top: 12px;
}

.detail-title {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-secondary);
  margin-bottom: 8px;
}

.char-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.char-row {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
}

.char-avatar {
  font-size: 18px;
  width: 24px;
  text-align: center;
}

.char-name {
  width: 48px;
  font-weight: 500;
  flex-shrink: 0;
}

.affinity-bar-wrap {
  flex: 1;
  height: 6px;
  background: var(--bg-secondary);
  border-radius: 3px;
  overflow: hidden;
}

.affinity-bar {
  height: 100%;
  border-radius: 3px;
  transition: width 0.3s;
}

.char-stage {
  font-size: 12px;
  font-weight: 500;
  width: 36px;
  text-align: center;
  flex-shrink: 0;
}

.char-affinity {
  font-size: 11px;
  color: var(--text-muted);
  width: 28px;
  text-align: right;
  flex-shrink: 0;
}

.event-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.event-tag {
  display: inline-block;
  padding: 2px 10px;
  border-radius: 9999px;
  background: var(--accent-light);
  color: var(--accent-primary);
  font-size: 12px;
  font-weight: 500;
}

.collection-info {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 10px;
  font-size: 13px;
}

.collection-total {
  font-weight: 600;
  color: var(--text-primary);
}

.collection-rarity {
  color: var(--text-secondary);
  font-size: 12px;
}

.collection-latest {
  color: var(--text-muted);
  font-size: 12px;
  font-style: italic;
}

.detail-actions {
  margin-top: 14px;
  display: flex;
  gap: 8px;
}

.action-btn {
  flex: 1;
  padding: 8px 0;
  border-radius: var(--radius-sm);
  font-size: 14px;
  font-weight: 500;
  text-align: center;
  transition: all 0.2s;
}

.load-btn {
  background: var(--accent-primary);
  color: white;
}

.load-btn:hover {
  filter: brightness(1.1);
}

.save-btn {
  background: var(--accent-light);
  color: var(--accent-primary);
  border: 1.5px solid var(--accent-primary);
}

.save-btn:hover {
  background: var(--accent-primary);
  color: white;
}

.empty-saves {
  text-align: center;
  padding: 30px;
  color: var(--text-muted);
  font-size: 14px;
}

.new-save-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 14px;
  background: var(--accent-light);
  border: 2px dashed var(--accent-primary);
  border-radius: var(--radius-md);
  cursor: pointer;
  color: var(--accent-primary);
  font-weight: 500;
  transition: all 0.2s;
}

.new-save-btn:hover {
  background: var(--accent-primary);
  color: white;
}

.plus-icon {
  font-size: 20px;
  font-weight: 700;
}
</style>
