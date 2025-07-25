<script setup>
import { computed, h, reactive, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import AddLink from '@/components/icons/AddLink.vue'
import { NButton, NIcon, useThemeVars } from 'naive-ui'
import { types, types as redisTypes } from '@/consts/support_redis_type.js'
import EditableTableColumn from '@/components/common/EditableTableColumn.vue'
import useDialogStore from 'stores/dialog.js'
import { isEmpty, size, truncate } from 'lodash'
import { decodeTypes, formatTypes } from '@/consts/value_view_type.js'
import useBrowserStore from 'stores/browser.js'
import LoadList from '@/components/icons/LoadList.vue'
import LoadAll from '@/components/icons/LoadAll.vue'
import IconButton from '@/components/common/IconButton.vue'
import ContentEntryEditor from '@/components/content_value/ContentEntryEditor.vue'
import Edit from '@/components/icons/Edit.vue'
import FormatSelector from '@/components/content_value/FormatSelector.vue'
import { decodeRedisKey, nativeRedisKey } from '@/utils/key_convert.js'
import ContentSearchInput from '@/components/content_value/ContentSearchInput.vue'
import { formatBytes } from '@/utils/byte_convert.js'
import copy from 'copy-text-to-clipboard'
import SwitchButton from '@/components/common/SwitchButton.vue'
import AlignLeft from '@/components/icons/AlignLeft.vue'
import AlignCenter from '@/components/icons/AlignCenter.vue'
import { TextAlignType } from '@/consts/text_align_type.js'

const i18n = useI18n()
const themeVars = useThemeVars()

const props = defineProps({
    name: String,
    db: Number,
    keyPath: String,
    keyCode: {
        type: Array,
        default: null,
    },
    ttl: {
        type: Number,
        default: -1,
    },
    value: {
        type: [String, Array],
        default: () => [],
    },
    size: Number,
    length: Number,
    format: String,
    decode: String,
    end: Boolean,
    loading: Boolean,
    textAlign: Number,
})

const emit = defineEmits(['loadmore', 'loadall', 'reload', 'match', 'update:textAlign'])

/**
 *
 * @type {ComputedRef<string|number[]>}
 */
const keyName = computed(() => {
    return !isEmpty(props.keyCode) ? props.keyCode : props.keyPath
})

const browserStore = useBrowserStore()
const dialogStore = useDialogStore()
const keyType = redisTypes.HASH
const currentEditRow = reactive({
    no: 0,
    key: '',
    value: null,
    format: formatTypes.RAW,
    decode: decodeTypes.NONE,
})

const inEdit = computed(() => {
    return currentEditRow.no > 0
})
const fullEdit = ref(false)

const tableRef = ref(null)
const fieldFilterOption = ref(null)
const fieldColumn = computed(() => ({
    key: 'key',
    title: () => i18n.t('common.field'),
    align: props.textAlign !== TextAlignType.Left ? 'center' : 'left',
    titleAlign: 'center',
    resizable: true,
    ellipsis: {
        tooltip: {
            style: {
                maxWidth: '50vw',
                maxHeight: '50vh',
            },
            scrollable: true,
        },
        lineClamp: 1,
    },
    filterOptionValue: fieldFilterOption.value,
    className: inEdit.value ? 'clickable wordline' : 'wordline',
    filter: (value, row) => {
        return !!~row.k.indexOf(value.toString())
    },
    render: (row) => {
        if (row.rm === true) {
            return h('s', {}, decodeRedisKey(row.k))
        }
        return decodeRedisKey(row.k)
    },
}))

const isCode = computed(() => {
    return props.format === formatTypes.JSON || props.format === formatTypes.UNICODE_JSON
})
// const valueFilterOption = ref(null)
const valueColumn = computed(() => ({
    key: 'value',
    title: () => i18n.t('common.value'),
    align: isCode.value ? 'left' : props.textAlign !== TextAlignType.Left ? 'center' : 'left',
    titleAlign: 'center',
    resizable: true,
    ellipsis: isCode.value
        ? false
        : {
              tooltip: {
                  style: {
                      maxWidth: '50vw',
                      maxHeight: '50vh',
                  },
                  scrollable: true,
              },
              lineClamp: 1,
          },
    // filterOptionValue: valueFilterOption.value,
    className: inEdit.value ? 'clickable' : '',
    // filter: (value, row) => {
    //     if (row.dv) {
    //         return !!~row.dv.indexOf(value.toString())
    //     }
    //     return !!~row.v.indexOf(value.toString())
    // },
    render: (row) => {
        let val = row.dv || nativeRedisKey(row.v)
        if (isCode.value) {
            return h('pre', { class: 'pre-wrap' }, val)
        }
        val = truncate(val, { length: 500 })
        if (row.rm === true) {
            return h('s', {}, val)
        }
        return val
    },
}))

const startEdit = async (no, key, value) => {
    currentEditRow.no = no
    currentEditRow.key = key
    currentEditRow.value = value
    currentEditRow.decode = props.decode
    currentEditRow.format = props.format
}

const saveEdit = async (field, value, decode, format) => {
    try {
        const row = props.value[currentEditRow.no - 1]
        if (row == null) {
            throw new Error('row not exists')
        }

        if (isEmpty(field)) {
            field = currentEditRow.key
        }

        const { success, msg } = await browserStore.setHash({
            server: props.name,
            db: props.db,
            key: keyName.value,
            field: row.k,
            newField: field,
            value,
            decode,
            format,
            retDecode: props.decode,
            retFormat: props.format,
            index: [currentEditRow.no - 1],
        })
        if (success) {
            currentEditRow.value = value
            $message.success(i18n.t('interface.save_value_succ'))
        } else {
            $message.error(msg)
        }
    } catch (e) {
        $message.error(e.message)
    }
}

const resetEdit = () => {
    currentEditRow.no = 0
    currentEditRow.key = ''
    currentEditRow.value = null
    // if (currentEditRow.format !== props.format || currentEditRow.decode !== props.decode) {
    //     nextTick(() => onFormatChanged(currentEditRow.decode, currentEditRow.format))
    // }
    // currentEditRow.format = formatTypes.RAW
    // currentEditRow.decode = decodeTypes.NONE
}

const actionColumn = {
    key: 'action',
    title: () => i18n.t('interface.action'),
    width: 120,
    align: 'center',
    titleAlign: 'center',
    fixed: 'right',
    render: (row, index) => {
        return h(EditableTableColumn, {
            editing: false,
            bindKey: row.k,
            canRefresh: true,
            onRefresh: async () => {
                const { updated, success, msg } = await browserStore.getHashField({
                    server: props.name,
                    db: props.db,
                    key: keyName.value,
                    field: row.k,
                    decode: props.decode,
                    format: props.format,
                })
                if (success) {
                    delete props.value[index]['rm']
                    $message.success(i18n.t('dialogue.reload_succ'))
                } else {
                    // update fail, the key may have been deleted
                    $message.error(msg)
                    props.value[index]['rm'] = true
                }
            },
            onCopy: async () => {
                copy(row.v)
                $message.success(i18n.t('interface.copy_succ'))
            },
            onEdit: () => startEdit(index + 1, row.k, row.v),
            onDelete: async () => {
                try {
                    const { removed, success, msg } = await browserStore.removeHashField({
                        server: props.name,
                        db: props.db,
                        key: keyName.value,
                        field: row.k,
                        reload: false,
                    })
                    if (success) {
                        props.value.splice(index, 1)
                        $message.success(i18n.t('dialogue.delete.success', { key: row.k }))
                    } else {
                        $message.error(msg)
                    }
                } catch (e) {
                    $message.error(e.message)
                }
            },
        })
    },
}

const columns = computed(() => {
    if (!inEdit.value) {
        return [
            {
                key: 'no',
                title: '#',
                width: 80,
                align: 'center',
                titleAlign: 'center',
                render: (row, index) => {
                    return index + 1
                },
            },
            fieldColumn.value,
            valueColumn.value,
            actionColumn,
        ]
    } else {
        return [
            {
                key: 'no',
                title: '#',
                width: 80,
                align: 'center',
                titleAlign: 'center',
                render: (row, index) => {
                    if (index + 1 === currentEditRow.no) {
                        // editing row, show edit state
                        return h(NIcon, { size: 16, color: 'red' }, () => h(Edit, { strokeWidth: 5 }))
                    } else {
                        return index + 1
                    }
                },
            },
            fieldColumn.value,
        ]
    }
})

const rowProps = (row, index) => {
    return {
        onClick: () => {
            // in edit mode, switch edit row by click
            if (inEdit.value) {
                startEdit(index + 1, row.k, row.v)
            }
        },
    }
}

const entries = computed(() => {
    const len = size(props.value)
    return `${len} / ${Math.max(len, props.length)}`
})

const loadProgress = computed(() => {
    const len = size(props.value)
    return (len * 100) / Math.max(len, props.length)
})

const showMemoryUsage = computed(() => {
    return !isNaN(props.size) && props.size > 0
})

const onAddRow = () => {
    dialogStore.openAddFieldsDialog(props.name, props.db, props.keyPath, props.keyCode, types.HASH)
}

const onFilterInput = (val) => {
    fieldFilterOption.value = val
}

const onMatchInput = (matchVal, filterVal) => {
    fieldFilterOption.value = filterVal
    emit('match', matchVal)
}

const onUpdateFilter = (filters, sourceColumn) => {
    fieldFilterOption.value = filters[sourceColumn.key]
}

const onFormatChanged = (selDecode, selFormat) => {
    emit('reload', selDecode, selFormat)
}

const searchInputRef = ref(null)
defineExpose({
    reset: () => {
        resetEdit()
        searchInputRef.value?.reset()
    },
})
</script>

<template>
    <div class="content-wrapper flex-box-v">
        <slot name="toolbar" />
        <div class="tb2 value-item-part flex-box-h">
            <div class="flex-box-h" style="max-width: 50%">
                <content-search-input
                    ref="searchInputRef"
                    @filter-changed="onFilterInput"
                    @match-changed="onMatchInput" />
            </div>
            <div class="flex-item-expand"></div>
            <switch-button
                :icons="[AlignCenter, AlignLeft]"
                :stroke-width="3.5"
                :t-tooltips="['interface.text_align_center', 'interface.text_align_left']"
                :value="props.textAlign"
                size="medium"
                unselect-stroke-width="3"
                @update:value="(val) => emit('update:textAlign', val)" />
            <n-divider vertical />
            <n-button-group>
                <icon-button
                    :disabled="props.end || props.loading"
                    :icon="LoadList"
                    border
                    size="18"
                    t-tooltip="interface.load_more_entries"
                    @click="emit('loadmore')" />
                <icon-button
                    :disabled="props.end || props.loading"
                    :icon="LoadAll"
                    border
                    size="18"
                    t-tooltip="interface.load_all_entries"
                    @click="emit('loadall')" />
            </n-button-group>
            <n-button :focusable="false" plain @click="onAddRow">
                <template #icon>
                    <n-icon :component="AddLink" size="18" />
                </template>
                {{ $t('interface.add_row') }}
            </n-button>
        </div>
        <!-- loaded progress -->
        <n-progress
            :border-radius="0"
            :color="props.end ? '#0000' : themeVars.primaryColor"
            :height="2"
            :percentage="loadProgress"
            :processing="props.loading"
            :show-indicator="false"
            status="success"
            type="line" />
        <div id="content-table" class="value-wrapper value-item-part flex-box-h flex-item-expand">
            <!-- table -->
            <n-data-table
                ref="tableRef"
                :bordered="false"
                :bottom-bordered="false"
                :columns="columns"
                :data="props.value"
                :loading="props.loading"
                :row-key="(row) => row.k"
                :row-props="rowProps"
                :single-column="true"
                :single-line="false"
                class="flex-item-expand"
                flex-height
                size="small"
                striped
                virtual-scroll
                @update:filters="onUpdateFilter" />

            <!-- edit pane -->
            <div
                v-show="inEdit"
                :style="{ position: fullEdit ? 'static' : 'relative' }"
                class="entry-editor-container flex-item-expand"
                style="width: 100%">
                <content-entry-editor
                    v-model:decode="currentEditRow.decode"
                    v-model:format="currentEditRow.format"
                    v-model:fullscreen="fullEdit"
                    :field="currentEditRow.key"
                    :field-label="$t('common.field')"
                    :key-path="props.keyPath"
                    :show="inEdit"
                    :value="currentEditRow.value"
                    :value-label="$t('common.value')"
                    class="flex-item-expand"
                    style="width: 100%"
                    @close="resetEdit"
                    @save="saveEdit" />
            </div>
        </div>
        <div class="value-footer flex-box-h">
            <n-text v-if="!isNaN(props.length)">{{ $t('interface.entries') }}: {{ entries }}</n-text>
            <n-divider v-if="showMemoryUsage" vertical />
            <n-text v-if="showMemoryUsage">{{ $t('interface.memory_usage') }}: {{ formatBytes(props.size) }}</n-text>
            <div class="flex-item-expand"></div>
            <format-selector
                v-show="!inEdit"
                :decode="props.decode"
                :disabled="inEdit"
                :format="props.format"
                @format-changed="onFormatChanged" />
        </div>
    </div>
</template>

<style lang="scss" scoped>
.value-footer {
    border-top: v-bind('themeVars.borderColor') 1px solid;
    background-color: v-bind('themeVars.tableHeaderColor');
}
</style>
