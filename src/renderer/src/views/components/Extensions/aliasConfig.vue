<template>
  <div class="alias-config">
    <a-card
      :bordered="false"
      class="alias-config-container responsive-table"
    >
      <a-row>
        <a-col :span="10">
          <a-input
            v-model:value="searchValue"
            :placeholder="t('extensions.fuzzySearch')"
            class="search-input"
            @search="handleTableChange"
            @press-enter="handleTableChange"
          >
            <template #suffix>
              <search-outlined />
            </template>
          </a-input>
        </a-col>
        <a-col :span="2">
          <div
            style="margin-left: 10px"
            @click="handleAdd"
          >
            <a-button
              type="primary"
              size="small"
              class="workspace-button"
              :icon="h(PlusOutlined)"
            >
              {{ t('extensions.addCommand') }}
            </a-button>
          </div>
        </a-col>
      </a-row>
      <a-table
        style="margin-top: 10px"
        :columns="columns"
        :data-source="list"
        :pagination="false"
        class="alias-config-table"
        @change="handleTableChange"
      >
        <template #emptyText>{{ t('common.noData') }}</template>
        <template #bodyCell="{ column, record }">
          <template v-if="column.key === 'alias'">
            <a-input
              v-model:value="record.alias"
              :disabled:="record.edit"
              :class="{ 'editable-input': record.edit }"
            />
          </template>
          <template v-else-if="column.key === 'command'">
            <a-textarea
              v-model:value="record.command"
              class="textarea-command"
              :auto-size="{ minRows: 1, maxRows: 5 }"
              :disabled:="record.edit"
              :class="{ 'editable-input': record.edit }"
              :spellcheck="false"
            />
          </template>
          <template v-else-if="column.key === 'action'">
            <a-button
              v-if="!record.edit"
              type="link"
              style="width: 40px"
              :icon="h(EditOutlined)"
              @click="columnOpt('edit', record)"
            >
            </a-button>
            <a-button
              v-if="!record.edit"
              type="link"
              style="width: 40px; color: #ff4d4f"
              :icon="h(CloseOutlined)"
              @click="columnOpt('del', record)"
            >
            </a-button>
            <a-button
              v-if="record.edit"
              type="link"
              style="width: 40px"
              :icon="h(CheckOutlined)"
              @click="columnOpt('save', record)"
            />
            <a-button
              v-if="record.edit"
              style="width: 40px"
              type="link"
              :icon="h(CloseSquareOutlined)"
              @click="columnOpt('cancel', record)"
            />
          </template>
        </template>
      </a-table>
    </a-card>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onBeforeUnmount, h } from 'vue'
import { PlusOutlined, CloseOutlined, EditOutlined, CheckOutlined, CloseSquareOutlined, SearchOutlined } from '@ant-design/icons-vue'
import 'xterm/css/xterm.css'
import { cloneDeep } from 'lodash'
import i18n from '@/locales'
import { notification } from 'ant-design-vue'
import { commandStore } from '@/services/commandStoreService' // 引入 commandStoreService
import { aliasConfigStore } from '@/store/aliasConfigStore'
import eventBus from '@/utils/eventBus'

const searchText = ref('')
const list = ref([])
const cloneRecord = ref({
  alias: '',
  command: '',
  edit: false,
  id: ''
})
const { t } = i18n.global
const columns = computed(() => [
  {
    title: t('extensions.alias'),
    dataIndex: 'alias',
    key: 'alias',
    width: '20%'
  },
  {
    title: t('extensions.command'),
    dataIndex: 'command',
    key: 'command'
  },
  {
    title: t('extensions.action'),
    dataIndex: 'action',
    key: 'action',
    width: '15%'
  }
])
const aliasStore = aliasConfigStore()
// 使用 CommandStoreService 处理数据
const handleTableChange = async () => {
  try {
    // 使用搜索功能获取过滤后的别名
    list.value = await commandStore.search(searchText.value)
  } catch (err) {
    console.error('Error loading aliases:', err)
    notification.error({
      message: t('extensions.error'),
      placement: 'topRight',
      duration: 3
    })
    list.value = []
  }
}

const columnOpt = async (type, record) => {
  switch (type) {
    case 'edit':
      // 处理编辑前的状态清理
      if (list.value.length > 0 && list.value[0]?.id === 'new') {
        list.value.splice(0, 1)
      } else if (cloneRecord.value.id && cloneRecord.value.id !== 'new') {
        const index = list.value.findIndex((item) => item.alias === cloneRecord.value.alias)
        if (index !== -1) {
          list.value[index].alias = cloneRecord.value.alias
          list.value[index].command = cloneRecord.value.command
          list.value[index].edit = false
        }
      }
      record.edit = true
      cloneRecord.value = cloneDeep(record)
      await aliasConfigRefresh()
      break

    case 'del':
      try {
        // 使用 alias 删除记录
        await commandStore.delete(record.alias)
        await handleTableChange()
        await aliasConfigRefresh()
        cloneRecordReset()
        notification.success({
          message: t('extensions.success'),
          placement: 'topRight',
          duration: 2
        })
      } catch (err) {
        console.error('Error deleting alias:', err)
        notification.error({
          message: t('extensions.error'),
          placement: 'topRight',
          duration: 3
        })
      }
      break

    case 'save':
      if (record.alias.length === 0 || record.command.length === 0) {
        notification.warning({
          message: t('extensions.warning'),
          description: t('extensions.missingAliasCommand'),
          placement: 'topRight',
          duration: 2
        })
        return
      }

      try {
        // 如果是编辑现有记录且别名发生了变更
        if (record.id && record.id !== 'new' && record.alias !== cloneRecord.value.alias) {
          // 检查新别名是否存在
          const existingAlias = await commandStore.getByAlias(record.alias)
          if (existingAlias) {
            notification.warning({
              message: t('extensions.warning'),
              description: t('extensions.aliasAlreadyExists'),
              placement: 'topRight',
              duration: 2
            })
            return
          }
          const recordNew = {
            id: record.id,
            alias: record.alias,
            command: record.command
          }
          // 需要重命名别名
          const success = await commandStore.renameAlias(cloneRecord.value.alias, recordNew)
          if (!success) {
            throw new Error('Failed to rename alias')
          }

          notification.success({
            message: t('extensions.success'),
            placement: 'topRight',
            duration: 2
          })
        } else if (record.id === 'new') {
          // 检查别名是否已存在
          const existingAlias = await commandStore.getByAlias(record.alias)
          if (existingAlias) {
            notification.warning({
              message: t('extensions.warning'),
              description: t('extensions.aliasAlreadyExists'),
              placement: 'topRight',
              duration: 2
            })
            return
          }

          // 新建别名
          const newId = Date.now().toString(36) + Math.random().toString(36).substring(2)
          await commandStore.add({
            id: newId,
            alias: record.alias,
            command: record.command
          })

          notification.success({
            message: t('extensions.success'),
            placement: 'topRight',
            duration: 2
          })
        } else {
          // 更新别名内容但不更改别名名称
          await commandStore.update({
            id: record.id,
            alias: record.alias,
            command: record.command
          })

          notification.success({
            message: t('extensions.success'),
            placement: 'topRight',
            duration: 2
          })
        }

        record.edit = false
        await handleTableChange()
        await aliasConfigRefresh()
        cloneRecordReset()
      } catch (err) {
        console.error('Error saving alias:', err)
        notification.error({
          message: t('extensions.error'),
          description: err,
          placement: 'topRight',
          duration: 3
        })
      }
      break

    case 'cancel':
      if (record.id === 'new') {
        // 移除新添加的空记录
        list.value.splice(0, 1)
      } else {
        // 恢复原始值
        record.alias = cloneRecord.value.alias
        record.command = cloneRecord.value.command
      }
      record.edit = false
      cloneRecordReset()
      break
  }
}

// 监听别名状态变化事件
onMounted(() => {
  // 初始加载数据
  handleTableChange()
  // 使用eventBus监听
  eventBus.on('aliasStatusChanged', () => {
    // 重新加载数据
    handleTableChange()
      .then(() => {})
      .catch((err) => {
        console.error('重新加载别名数据失败:', err)
      })
  })
})

// 移除事件监听
onBeforeUnmount(() => {
  eventBus.off('aliasStatusChanged')
})

const cloneRecordReset = () => {
  cloneRecord.value = {
    alias: '',
    command: '',
    edit: false,
    id: ''
  }
}

const handleAdd = () => {
  // 检查是否已有正在编辑的新记录
  const existingNewRecord = list.value.find((item) => item.id === 'new')
  if (existingNewRecord) {
    return // 如果已有新记录正在编辑，不再添加
  }

  // 新的记录 ID 使用特殊标识 'new'
  const newRecord = {
    alias: '',
    command: '',
    edit: true,
    id: 'new'
  }

  if (list.value.length) {
    list.value = [newRecord, ...list.value]
  } else {
    list.value.push(newRecord)
  }

  // 如果有其他正在编辑的记录，先恢复其状态
  if (cloneRecord.value.edit === true && cloneRecord.value.id && cloneRecord.value.id !== 'new') {
    const index = list.value.findIndex((item) => item.alias === cloneRecord.value.alias)
    if (index !== -1) {
      list.value[index].alias = cloneRecord.value.alias
      list.value[index].command = cloneRecord.value.command
      list.value[index].edit = false
    }
  }

  cloneRecord.value = {
    alias: '',
    command: '',
    edit: true,
    id: 'new'
  }
}

// 刷新别名配置 - 在本地存储环境下，主要是触发其他使用别名的组件刷新
const aliasConfigRefresh = async () => {
  await aliasStore.refreshAliasesFromDB()
}
</script>

<style lang="less" scoped>
.alias-config {
  width: 100%;
  height: 100%;
}

.alias-config-container {
  width: 100%;
  height: 100%;
  background-color: #141414;
  border-radius: 6px;
  padding: 4px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0);
  color: #ffffff;
}

.search-input {
  background-color: #2c2c2c;
  border-color: #2c2c2c;
  color: white;
  :deep(.ant-input) {
    color: rgba(255, 255, 255, 0.85);
    background-color: transparent !important; /* 设置背景透明 */
    border-color: rgba(255, 255, 255, 0.2) !important;

    &::placeholder {
      color: rgba(255, 255, 255, 0.5);
    }
  }
}

.workspace-button {
  font-size: 14px;
  height: 30px;
  display: flex;
  align-items: center;
  background-color: #1677ff;
  border-color: #1677ff;

  &:hover {
    background-color: #398bff;
    border-color: #398bff;
  }

  &:active {
    background-color: rgb(130, 134, 155);
    border-color: rgb(130, 134, 155);
  }
}

:deep(.ant-card) {
  height: 100%;
}

:deep(.ant-card-body) {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.alias-config-table :deep(.ant-table-tbody) {
  background-color: #141414; /* 黑色背景 */
}

.alias-config-table :deep(.ant-table-thead > tr > th) {
  background: #3a3a3a; /* 深灰色表头 */
  color: #fff;
  padding: 8px;
  border-radius: 0;
  border: none !important;
  border-bottom: 1px solid #4a4a4a !important; /* 可选：保留表头底部边框 */
}

.alias-config-table :deep(.ant-table-tbody > tr:nth-child(even) > td) {
  /* 可以选择保留默认样式或自定义 */
  /* 例如: */
  background: #141414;
  color: #fff;
  padding: 8px;
  border: none !important;
}

.alias-config-table :deep(.ant-table-tbody > tr:nth-child(odd) > td) {
  /* 可以选择保留默认样式或自定义 */
  /* 例如: */
  background: #1f1f1f;
  color: #fff;
  padding: 8px;
  border: none !important;
}

.alias-config-table :deep(.ant-input) {
  background-color: transparent; /* 透明背景 */
  border: none; /* 移除边框 */
  box-shadow: none; /* 移除阴影 */
  color: #fff; /* 输入文字为白色 */
}

.alias-config-table :deep(.ant-table-container table > thead > tr:first-child > *:last-child) {
  border-start-end-radius: 0 !important;
  border-end-end-radius: 0 !important;
}

.alias-config-table :deep(.ant-table-container table > thead > tr:first-child > *:first-child) {
  border-start-start-radius: 0 !important;
  border-end-start-radius: 0 !important;
}

.alias-config-table {
  /* 可编辑状态的输入框样式 - 灰色背景 */
  .editable-input {
    background-color: #3a3a3a !important; /* 灰色背景 */
    padding: 4px 8px;
    border-radius: 4px;
  }

  .textarea-command {
    white-space: pre-wrap;
  }
}

/* 处理滚动条轨道和滑块 */
.alias-config-table .textarea-command::-webkit-scrollbar-track,
.alias-config-table .textarea-command::-webkit-scrollbar-thumb,
.alias-config-table .textarea-command :deep(.ant-input::-webkit-scrollbar-track),
.alias-config-table .textarea-command :deep(.ant-input::-webkit-scrollbar-thumb),
.alias-config-table :deep(.textarea-command .ant-input::-webkit-scrollbar-track),
.alias-config-table :deep(.textarea-command .ant-input::-webkit-scrollbar-thumb) {
  display: none !important;
  background: transparent !important;
}

.responsive-table {
  width: 100%;
  margin-left: auto;
  margin-right: auto;
}

.alias-config-table:deep(.ant-table-tbody > tr:hover > td) {
  background-color: #4a4a4a !important; /* 浅灰色悬浮背景 */
}
</style>
