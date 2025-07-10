<template>
  <vue-nestable
      :value="value"
      :max-depth="maxDepth"
      @input="value = $event"
      @change="onMenuChange"
      :hooks="{
      beforeMove: beforeMove,
    }"
      class="px-3"
      classProp="classProp"
  >
    <template v-slot="{ item }">
      <vue-nestable-handle
          class="handle dark:o1-bg-gray-800 o1-flex io1-tems-center o1-justify-between o1-border o1-rounded-lg o1-outline-none o1-border-b o1-border-gray-200 dark:o1-border-gray-600"
      >
        <div class="item-data o1-items-center o1-flex o1-shrink o1-min-w-0" :class="{ 'o1-px-3': !hasChildren(item) }">
          <button
              v-if="hasChildren(item)"
              @click.prevent="toggleMenuChildrenCascade(item)"
              class="o1-appearance-none o1-cursor-pointer o1-fill-current hover:o1-text-primary o1-flex o1-px-3 o1-items-center focus:o1-outline-none"
          >
            <Icon :name="getIconName(item)" type="outline" />
          </button>

          <div
              class="o1-text-90 o1-font-bold o1-whitespace-nowrap o1-overflow-hidden o1-text-ellipsis"
              :class="{ 'opacity-25': !item.enabled }"
          >
            {{ item.name }}
          </div>

          <div
              class="o1-font-lighter o1-text-80 o1-ml-4 o1-text-sm o1-whitespace-nowrap o1-text-ellipsis o1-overflow-hidden"
              :class="{ 'o1-opacity-25': !item.enabled }"
          >
            {{ item.displayValue }}
          </div>
        </div>

        <div class="buttons o1-gap-x-2 md:o1-w-1/3 o1-flex o1-justify-end o1-content-center">
          <span class="mt-2 text-gray-500 dark:text-gray-400">
            <Icon v-if="item.exception" name="x-circle" type="solid" class="text-red-500" />
          </span>

          <span
              :title="__('novaMenuBuilder.edit')"
              @click.prevent="$emit('editMenu', item)"
              class="mt-2 cursor-pointer text-gray-500 dark:text-gray-400 hover:[&:not(:disabled)]:text-primary-500 dark:hover:[&:not(:disabled)]:text-primary-500"
          >
            <Icon name="PencilAlt" type="outline" />
          </span>

          <span
              v-if="this.copyable"
              :title="__('novaMenuBuilder.duplicate')"
              @click.prevent="$emit('duplicateMenuItem', item)"
              class="mt-2 cursor-pointer text-gray-500 dark:text-gray-400 hover:[&:not(:disabled)]:text-primary-500 dark:hover:[&:not(:disabled)]:text-primary-500"
          >
            <Icon name="Duplicate" type="outline" />
          </span>

          <span
              :title="__('novaMenuBuilder.delete')"
              @click.prevent="!item.enabled && $emit('removeMenu', item)"
              :class="{'disabled': item.enabled}"
              class="mt-2 cursor-pointer text-gray-500 dark:text-gray-400 hover:[&:not(:disabled)]:text-primary-500 dark:hover:[&:not(:disabled)]:text-primary-500"
          >
            <Icon name="trash" type="outline" />
          </span>
        </div>
      </vue-nestable-handle>
    </template>
  </vue-nestable>
</template>

<script>
import { VueNestable, VueNestableHandle } from 'vue3-nestable';
import { Icon } from 'laravel-nova-ui';

export default {
  props: {
    value: {
      type: Array,
      required: true,
    },
    maxDepth: {
      type: Number,
      required: false,
      default: 10,
    },
  },
  computed: {
    getIconName() {
      return item => (this.isCascadeOpen(item) ? 'SortDescending' : 'ViewList');
    },
    copyable() {
      return Nova.config('copyable');
    }
  },
  components: {
    VueNestable,
    VueNestableHandle,
    Icon,
  },

  data: () => ({
    duplicateItem: null,
    placeholderItems: [],
    isDragging: false,
  }),

  methods: {
    hasChildren(item) {
      return Array.isArray(item.children) && item.children.length;
    },

    toggleMenuChildrenCascade(item) {
      if (item.classProp.find(className => className === 'hide-cascade')) {
        item.classProp.splice(item.classProp.indexOf('hide-cascade'), 1);
      } else {
        item.classProp.push('hide-cascade');
      }
      this.$emit('saveMenuLocalState', item);
    },

    isCascadeOpen(item) {
      return !item.classProp.find(className => className === 'hide-cascade');
    },

    // 创建占位符项
    createPlaceholder() {
      return {
        id: `placeholder-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
        name: 'Drag here',
        displayValue: '',
        enabled: false,
        children: [],
        classProp: ['placeholder-item'],
        isPlaceholder: true,
        nestable: true
      };
    },

    // 添加占位符到指定层级
    addPlaceholders(dragItem) {
      let targetLevels = [];

      if (dragItem?.allowed_level && Array.isArray(dragItem.allowed_level)) {
        targetLevels = dragItem.allowed_level.map(level => level - 1).filter(level => level > 0);
      } else {
        targetLevels = [1];
      }

      const addPlaceholdersToLevel = (items, currentLevel) => {
        items.forEach(item => {
          if (targetLevels.includes(currentLevel) && !this.hasChildren(item)) {
            if (!item.children) item.children = [];

            // ✅ 避免重复插入占位符
            if (!item.children.find(child => child.isPlaceholder)) {
              const placeholder = this.createPlaceholder();
              item.children.push(placeholder);
              this.placeholderItems.push({
                parentItem: item,
                placeholder,
                level: currentLevel
              });
            }
          }

          if (item.children && item.children.length > 0) {
            const realChildren = item.children.filter(child => !child.isPlaceholder);
            if (realChildren.length > 0) {
              addPlaceholdersToLevel(realChildren, currentLevel + 1);
            }
          }
        });
      };

      addPlaceholdersToLevel(this.value, 1);
    },

    // 移除所有占位符
    removePlaceholders() {
      // 创建一个新的数组，递归移除所有占位符
      const removeFromItems = (items) => {
        return items.map(item => {
          const newItem = { ...item };
          if (newItem.children && newItem.children.length > 0) {
            // 过滤掉占位符
            newItem.children = newItem.children.filter(child => !child.isPlaceholder);
            // 递归处理子项
            newItem.children = removeFromItems(newItem.children);
          }
          return newItem;
        });
      };

      const newValue = removeFromItems(this.value);
      this.placeholderItems = [];

      // 触发响应式更新，通知父组件数据变化
      this.$emit('input', newValue);
    },

    beforeMove({ dragItem, pathFrom, pathTo }) {
      if (dragItem?.allowed_level) {
        if (!this.isDragging) {
          this.isDragging = true;
          this.addPlaceholders(dragItem);
        }
        return dragItem.allowed_level.includes(pathTo.length);
      }

      return dragItem.nestable || pathTo.length === 1;
    },

    // 处理菜单变化事件
    onMenuChange() {
      // 当菜单发生变化时，清除占位符
      if (this.isDragging) {
        this.onDragEnd();
      }
      // 向父组件发送事件
      this.$emit('onMenuChange');
    },

    // 监听拖拽结束事件
    onDragEnd() {
      if (this.isDragging) {
        this.isDragging = false;
        // 使用nextTick确保DOM更新完成后再清除占位符
        this.$nextTick(() => {
          this.removePlaceholders();
        });
      }
    },
  },

  mounted() {
    // 添加额外的安全检查，确保占位符能被清除
    document.addEventListener('mouseup', this.onDragEnd);
    document.addEventListener('dragend', this.onDragEnd);
  },

  beforeUnmount() {
    // 清理事件监听器
    document.removeEventListener('mouseup', this.onDragEnd);
    document.removeEventListener('dragend', this.onDragEnd);

    // 组件销毁时清理占位符
    this.removePlaceholders();
  },
};
</script>

<style lang="scss">
.menu-builder {
  .v-popover,
  .v-popover > * > span {
    display: flex;
    justify-content: center;
    align-items: center;
  }
}
.justify-end {
  justify-content: flex-end;
}
.opacity-25 {
  opacity: 0.25;
}

// 占位符样式
.placeholder-item {
  opacity: 0;
  height: 3px;
  overflow: hidden;
  border: none;
}
</style>