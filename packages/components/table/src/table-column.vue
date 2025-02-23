<template>
  <template v-for="(item, index) in columns" :key="getKey(item)">
    <el-table-column
      :class-name="'plus-table-column ' + (hasPropsEditIcon ? 'plus-table-column__edit' : '')"
      :index="index"
      v-bind="item.tableColumnProps"
      :prop="item.prop"
      :width="item.width"
      :min-width="item.minWidth"
    >
      <template #header="scoped">
        <span class="plus-table-column__header">
          <PlusRender
            v-if="item.renderHeader && isFunction(item.renderHeader)"
            :render="item.renderHeader"
            :params="{ ...scoped, ...item, cellIndex: index }"
            :callback-value="getLabel(item.label)"
          />

          <!--表格单元格Header的插槽 -->
          <slot
            v-else
            :name="getTableHeaderSlotName(item.prop)"
            :prop="item.prop"
            :label="getLabel(item.label)"
            :field-props="item.fieldProps"
            :value-type="item.valueType"
            :cell-index="index"
            v-bind="scoped"
            :column="{ ...scoped, ...item }"
          >
            {{ getLabel(item.label) }}
          </slot>

          <el-tooltip v-if="item.tooltip" placement="top" v-bind="getTooltip(item.tooltip)">
            <slot name="tooltip-icon">
              <el-icon class="plus-table-column__header__icon" :size="16">
                <QuestionFilled />
              </el-icon>
            </slot>
          </el-tooltip>
        </span>
      </template>

      <template #default="{ row, column, $index, ...rest }">
        <template v-if="item.children?.length">
          <PlusTableColumn
            :columns="item.children"
            :editable="editable"
            @formChange="handleFormChange"
          />
          {{ item.label }}
        </template>

        <PlusDisplayItem
          v-else
          ref="plusDisplayItemInstance"
          :column="item"
          :row="row"
          :index="$index"
          :editable="editable"
          :rest="{ column, ...rest }"
          @change="data => handleChange(data, $index, column, item, rest)"
        >
          <!--表单单项的插槽 -->
          <template
            v-if="$slots[getFieldSlotName(item.prop)]"
            #[getFieldSlotName(item.prop)]="data"
          >
            <slot :name="getFieldSlotName(item.prop)" v-bind="data" />
          </template>

          <!-- 表单el-form-item 下一行额外的内容 的插槽 -->
          <template
            v-if="$slots[getExtraSlotName(item.prop)]"
            #[getExtraSlotName(item.prop)]="data"
          >
            <slot :name="getExtraSlotName(item.prop)" v-bind="data" />
          </template>

          <!--表格单元格的插槽 -->
          <template
            v-if="$slots[getTableCellSlotName(item.prop)]"
            #[getTableCellSlotName(item.prop)]="data"
          >
            <slot :name="getTableCellSlotName(item.prop)" v-bind="data" />
          </template>

          <!--表格单元格编辑的插槽 -->
          <template v-if="$slots['edit-icon']" #edit-icon>
            <slot name="edit-icon" />
          </template>
        </PlusDisplayItem>
      </template>
    </el-table-column>
  </template>
</template>

<script lang="ts" setup>
import { PlusDisplayItem } from '@plus-pro-components/components/display-item'
import type { PlusDisplayItemInstance } from '@plus-pro-components/components/display-item'
import type { PlusColumn, RecordType } from '@plus-pro-components/types'
import {
  getTooltip,
  getTableKey,
  getTableCellSlotName,
  getTableHeaderSlotName,
  getFieldSlotName,
  getExtraSlotName,
  isFunction,
  getLabel
} from '@plus-pro-components/components/utils'
import { TableFormRefInjectionKey } from '@plus-pro-components/constants'
import { QuestionFilled } from '@element-plus/icons-vue'
import type { Ref, ComputedRef } from 'vue'
import { ref, inject, watch, computed } from 'vue'
import { PlusRender } from '@plus-pro-components/components/render'
import type { TableColumnCtx } from 'element-plus'
import { ElTableColumn, ElTooltip, ElIcon } from 'element-plus'
import type { TableFormRefRow, FormChangeCallBackParams } from './type'

export interface PlusTableColumnProps {
  columns?: PlusColumn[]
  editable?: boolean | 'click' | 'dblclick'
}
export interface PlusTableColumnEmits {
  (e: 'formChange', data: FormChangeCallBackParams): void
}

defineOptions({
  name: 'PlusTableColumn'
})

const props = withDefaults(defineProps<PlusTableColumnProps>(), {
  columns: () => [],
  editable: false
})
const emit = defineEmits<PlusTableColumnEmits>()

/**
 *  表单ref处理
 */
const plusDisplayItemInstance = ref<PlusDisplayItemInstance[] | null>()
const formRefs = inject(TableFormRefInjectionKey) as Ref<Record<string | number, TableFormRefRow[]>>

/**
 *  设置表单ref
 */
const setFormRef = () => {
  if (!plusDisplayItemInstance.value?.length) return
  const data: RecordType = {}
  const list: { index: number; prop: string; formInstance: ComputedRef<any> }[] =
    plusDisplayItemInstance.value?.map(item => ({ ...item, ...item?.getDisplayItemInstance() })) ||
    []
  list.forEach(item => {
    if (!data[item.index]) {
      data[item.index] = []
    }
    data[item.index].push(item)
  })

  formRefs.value = data
}

watch(
  plusDisplayItemInstance,
  () => {
    setFormRef()
  },
  {
    deep: true,
    flush: 'post'
  }
)

// 是否需要editIcon
const hasPropsEditIcon = computed(() => props.editable === 'click' || props.editable === 'dblclick')

/**
 * 获取key
 * @param item
 */
// eslint-disable-next-line @typescript-eslint/ban-ts-comment
// @ts-ignore
const getKey = (item: PlusColumn) => getTableKey(item, true)

/**
 * 表单发生变化
 * @param data
 * @param index
 * @param column
 * @param item
 */
const handleChange = (
  data: { value: any; prop: string; row: RecordType },
  index: number,
  column: TableColumnCtx<RecordType>,
  item: PlusColumn,
  rest: RecordType
) => {
  const formChangeCallBackParams = {
    ...data,
    index,
    column: { ...column, ...item },
    rowIndex: index,
    ...rest
  } as unknown as FormChangeCallBackParams

  emit('formChange', formChangeCallBackParams)
}

const handleFormChange = (data: FormChangeCallBackParams) => {
  emit('formChange', data)
}

defineExpose({
  plusDisplayItemInstance
})
</script>
