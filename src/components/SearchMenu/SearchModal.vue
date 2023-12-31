<script lang="ts" setup>
import { computed, ref, shallowRef } from 'vue'
import {
  type RouteRecordName,
  type RouteRecordRaw,
  useRouter,
} from 'vue-router'
import { useAppStore } from '@/store/modules/app'
import { usePermissionStore } from '@/store/modules/permission'
import SearchResult from './SearchResult.vue'
import SearchFooter from './SearchFooter.vue'
import { ElScrollbar } from 'element-plus'
import { cloneDeep, debounce } from 'lodash-es'
import { DeviceEnum } from '@/constants/app-key'

interface Props {
  /** 控制 modal 显隐 */
  modelValue: boolean
}

const props = defineProps<Props>()
const emit = defineEmits<{
  'update:modelValue': [boolean]
}>()

const appStore = useAppStore()
const router = useRouter()

const inputRef = ref<HTMLInputElement | null>(null)
const scrollbarRef = ref<InstanceType<typeof ElScrollbar> | null>(null)
const searchResultRef = ref<InstanceType<typeof SearchResult> | null>(null)

const keyword = ref('')
const resultList = shallowRef<RouteRecordRaw[]>([])
const activeRouteName = ref<RouteRecordName>('')

/** 控制搜索对话框宽度 */
const modalWidth = computed(() =>
  appStore.device === DeviceEnum.Mobile ? '80vw' : '40vw'
)
/** 控制搜索对话框显隐 */
const modalVisible = computed({
  get() {
    return props.modelValue
  },
  set(value: boolean) {
    emit('update:modelValue', value)
  },
})
/** 树形菜单 */
const menusData = computed(() => cloneDeep(usePermissionStore().routes))

/** 搜索（防抖） */
const handleSearch = debounce(() => {
  const flatMenusData = flatTree(menusData.value)
  resultList.value = flatMenusData.filter((menu) =>
    keyword.value
      ? menu.meta?.title
          ?.toLocaleLowerCase()
          .includes(keyword.value.toLocaleLowerCase().trim())
      : false
  )
  // 默认选中搜索结果的第一项
  const length = resultList.value?.length
  activeRouteName.value = length > 0 ? resultList.value[0].name! : ''
}, 500)

/** 将树形菜单扁平化为一维数组，用于菜单搜索 */
const flatTree = (arr: RouteRecordRaw[], result: RouteRecordRaw[] = []) => {
  arr.forEach((item) => {
    result.push(item)
    item.children && flatTree(item.children, result)
  })
  return result
}

/** 关闭搜索对话框 */
const handleClose = () => {
  modalVisible.value = false
  // 延时处理防止用户看到重置数据的操作
  setTimeout(() => {
    keyword.value = ''
    resultList.value = []
  }, 200)
}

/** 根据下标位置进行滚动 */
const scrollTo = (index: number) => {
  const scrollTop = searchResultRef.value?.getScrollTop(index)
  // 手动控制 el-scrollbar 滚动条滚动，设置滚动条到顶部的距离
  scrollTop && scrollbarRef.value?.setScrollTop(scrollTop)
}

/** 键盘上键 */
const handleUp = () => {
  const { length } = resultList.value
  if (length === 0) return
  const index = resultList.value.findIndex(
    (item) => item.name === activeRouteName.value
  )
  if (index === 0) {
    // 如果已处在顶部，跳转到底部
    activeRouteName.value = resultList.value[length - 1].name!
    scrollTo(resultList.value.length - 1)
  } else {
    activeRouteName.value = resultList.value[index - 1].name!
    scrollTo(index - 1)
  }
}

/** 键盘下键 */
const handleDown = () => {
  const { length } = resultList.value
  if (length === 0) return
  const index = resultList.value.findIndex(
    (item) => item.name === activeRouteName.value
  )
  if (index === length - 1) {
    // 如果已处在底部，跳转到顶部
    activeRouteName.value = resultList.value[0].name!
    scrollTo(0)
  } else {
    activeRouteName.value = resultList.value[index + 1].name!
    scrollTo(index + 1)
  }
}

/** 键盘回车键 */
const handleEnter = () => {
  const { length } = resultList.value
  if (length === 0 || !activeRouteName.value) return
  router.push({ name: activeRouteName.value })
  handleClose()
}
</script>

<template>
  <el-dialog
    v-model="modalVisible"
    @opened="inputRef?.focus()"
    @closed="inputRef?.blur()"
    @keydown.up="handleUp"
    @keydown.down="handleDown"
    @keydown.enter="handleEnter"
    :before-close="handleClose"
    :width="modalWidth"
    top="5vh"
    class="search-modal__private"
    append-to-body
  >
    <el-input
      ref="inputRef"
      v-model="keyword"
      @input="handleSearch"
      placeholder="搜索菜单"
      size="large"
      clearable
    >
      <template #prefix>
        <SvgIcon name="search" />
      </template>
    </el-input>
    <el-empty
      v-if="resultList.length === 0"
      description="暂无搜索结果"
      :image-size="100"
    />
    <template v-else>
      <p>搜索结果</p>
      <el-scrollbar ref="scrollbarRef" max-height="40vh">
        <SearchResult
          ref="searchResultRef"
          v-model="activeRouteName"
          :list="resultList"
          @click="handleEnter"
        />
      </el-scrollbar>
    </template>
    <template #footer>
      <SearchFooter :total="resultList.length" />
    </template>
  </el-dialog>
</template>

<style lang="scss">
.search-modal__private {
  .svg-icon {
    font-size: 18px;
  }
  .el-dialog__header {
    display: none;
  }
  .el-dialog__footer {
    border-top: 1px solid var(--el-border-color);
    padding: var(--el-dialog-padding-primary);
  }
}
</style>
