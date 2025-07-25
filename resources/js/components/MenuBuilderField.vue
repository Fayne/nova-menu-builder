<template>
  <div id="menu-builder-field" class="relative py-3 o1-w-full">
    <menu-builder-header
      :locales="field.locales"
      :resourceId="resourceId"
      :activeLocale="selectedLocale"
      :menuCount="field.menuCount"
      :showDuplicate="field.showDuplicate"
      @addMenuItem="openAddModal"
      @changeLocale="setSelectedLocale"
      @refreshItems="refreshData"
    />

    <div class="py-6" v-if="loadingMenuItems">
      <loader class="text-60" />
    </div>

    <no-menu-items-placeholder @onAddClick="openAddModal" v-if="!loadingMenuItems && !menuItems.length" />

    <menu-builder
      v-if="!loadingMenuItems && menuItems.length"
      @duplicateMenuItem="duplicateMenuItem"
      @editMenu="editMenu"
      @onMenuChange="updateMenu"
      @removeMenu="removeMenu"
      @saveMenuLocalState="saveMenuLocalState"
      :max-depth="field.maxDepth"
      :value="menuItems"
      @input="menuItems = $event"
    />

    <update-menu-item-modal
      :max-depth="field.maxDepth"
      :max-level="maxLevel"
      :linkType="linkType"
      :menuItemTypes="menuItemTypes"
      :newItem="newItem"
      :resourceId="resourceId"
      :resourceName="resourceName"
      :showModal="showAddModal"
      :update="update"
      :errors="errors"
      :isMenuItemUpdating="isMenuItemUpdating"
      @closeModal="closeModal"
      @confirmItemCreate="confirmItemCreate"
      @onLinkModelUpdate="updateLinkModel"
      @onLinkTypeUpdate="updateLinkType"
      @updateItem="debouncedUpdateMenu"
    />

    <delete-menu-item-modal
      :itemToDelete="itemToDelete"
      :showModal="showDeleteModal"
      @closeModal="closeModal"
      @confirmItemDelete="confirmItemDelete"
    />
  </div>
</template>

<script>
import api from '../api';
import debounce from 'lodash/debounce';
import {FormField} from 'laravel-nova';
import MenuBuilderHeader from './core/MenuBuilderHeader';
import UpdateMenuItemModal from './modals/UpdateMenuItemModal';
import DeleteMenuItemModal from './modals/DeleteMenuItemModal';
import NoMenuItemsPlaceholder from './core/NoMenuItemsPlaceholder';
import HandlesCollapsibleState from '../mixins/HandlesCollapsibleState';

export default {
  mixins: [FormField, HandlesCollapsibleState],

  props: ['resourceName', 'resourceId', 'field'],

  components: {
    MenuBuilderHeader,
    NoMenuItemsPlaceholder,
    DeleteMenuItemModal,
    UpdateMenuItemModal,
  },

  data: () => ({
    loadingMenuItems: false,
    isMenuItemUpdating: false,
    selectedLocale: void 0,
    showDeleteModal: false,
    showAddModal: false,
    itemToDelete: null,
    update: false,
    linkType: {},
    errors: {},
    newItem: {
      name: null,
      value: '',
      target: '_self',
      menu_id: null,
      enabled: false,
      classProp: [],
    },
    menuItems: [],
    menuItemTypes: void 0,
  }),

  beforeMount() {
    // Set starting locale
    this.selectedLocale = Object.keys(this.field.locales)[0];
  },

  async mounted() {
    // Fix classes on Detail view
    this.$parent.$el.classList.remove('py-3', 'px-6');

    this.refreshData();
  },

  created() {
    this.debouncedUpdateMenu = debounce(this.updateMenu, 500);
  },

  beforeUnmount() {
    if (this.debouncedUpdateMenu?.cancel) {
      this.debouncedUpdateMenu.cancel();
    }
  },

  computed: {
    newItemData() {
      return {
        ...this.newItem,
        locale: this.selectedLocale,
        class: this.linkType.class,
        menu_id: this.resourceId,
      };
    },

    maxLevel() {
      if (this.menuItems.length === 0) {
        return 1;
      }

      for (const item of this.menuItems) {
        if (item.children?.length) {
          return 3;
        }
      }

      return 2;
    }
  },

  methods: {
    setSelectedLocale(locale) {
      this.selectedLocale = locale;
      this.refreshData();
    },

    openAddModal() {
      this.update = false;
      this.showAddModal = true;
    },

    closeModal() {
      this.showAddModal = false;
      this.showDeleteModal = false;
      this.resetNewItem();
    },

    async refreshData() {
      this.loadingMenuItems = true;

      const menuItems = (await api.getItems(this.resourceId, this.selectedLocale)).data;
      this.menuItems = this.setMenuItemProperties(
        Object.values(menuItems),
        this.getMenuLocalState(),
        this.field.collapsedAsDefault
      );

      const menuItemTypes = (await api.getMenuItemTypes(this.resourceId, this.selectedLocale)).data;
      this.menuItemTypes = Object.values(menuItemTypes);

      this.loadingMenuItems = false;
    },

    async editMenu(item) {
      if (Nova.config('edit_mode') === 'popup') {
        this.update = true;
        this.newItem = (await api.getMenuItem(item.id)).data;
        this.showAddModal = true;
        this.linkType = this.menuItemTypes.find(lt => lt.class === this.newItem.class) || {};
      } else {
        Nova.visit(`/resources/nova-menu-items/${item.id}/edit`);
      }
    },

    removeMenu(item) {
      this.itemToDelete = item;
      this.showDeleteModal = true;
    },

    async confirmItemDelete() {
      try {
        await api.destroy(this.itemToDelete.id);
        await this.refreshData();
        Nova.success(this.__('novaMenuBuilder.toastDeleteSucces'));
        this.itemToDelete = null;
        this.showDeleteModal = false;
      } catch (e) {
        this.handleErrors(e);
      }
    },

    resetNewItem() {
      this.errors = {};

      this.newItem = {
        name: null,
        value: '',
        target: '_self',
        enabled: false,
        menu_id: this.resourceId,
      };

      this.linkType = {};
    },

    async confirmItemCreate() {
      try {
        this.errors = {};

        await api.create(this.newItemData);
        this.refreshData();
        this.showAddModal = false;
        this.resetNewItem();
        Nova.success(this.__('novaMenuBuilder.toastCreateSuccess'));
      } catch (e) {
        console.error(e);
        this.handleErrors(e);
      }
    },

    async updateItem() {
      try {
        this.isMenuItemUpdating = true;
        this.errors = {};
        await api.update(this.newItem.id, this.newItemData);
        this.isMenuItemUpdating = false;
        this.showAddModal = false;
        Nova.success(this.__('novaMenuBuilder.toastUpdateSuccess'));
        this.resetNewItem();
        await this.refreshData();
      } catch (e) {
        this.isMenuItemUpdating = false;
        this.handleErrors(e);
      }
    },

    async updateMenu() {
      try {
        await api.saveItems(this.resourceId, this.menuItems);
        Nova.success(this.__('novaMenuBuilder.toastReorderSuccess'));
      } catch (e) {
        const message = e?.response?.data?.message;
        if (message) {
          Nova.error(message);
        } else {
          Nova.error(this.__('novaMenuBuilder.serverError'));
        }
        this.refreshData();
      }
    },

    handleErrors(res) {
      let errors = res.response && res.response.data && res.response.data.errors;
      if (errors) {
        this.errors = errors;
        Object.values(errors).map(error => Nova.error(error));
      }
    },

    async duplicateMenuItem(item) {
      try {
        await api.duplicate(item.id);
        await this.refreshData();
        this.resetNewItem();
        Nova.success(this.__('novaMenuBuilder.toastDuplicateSuccess'));
      } catch (e) {
        this.handleErrors(e);
      }
    },

    updateLinkModel(modelId) {
      this.newItem.value = modelId || '';
    },

    updateLinkType(event) {
      const typeClass = event?.target?.value || event;
      const selectedType = this.menuItemTypes.find(type => type.class === typeClass) || null;

      if (selectedType) {
        this.linkType = Object.assign({}, selectedType);
      }

      this.newItem.value = '';
    },
  },
};
</script>

<style lang="scss">
[dusk='nova-menus-detail-component'] #menu-builder-field {
  margin: -8px 0;
}
#menu-builder-field {
  .menu-button {
    position: absolute;
    right: -12px;
    margin-top: -72px;
  }

  .nestable {
    position: relative;

    .nestable-list {
      margin: 0;
      padding: 0 0 0 40px;
      list-style-type: none;
    }

    > .nestable-list {
      padding: 0;
    }

    .nestable-item,
    .nestable-item-copy {
      margin: 10px 0 0;
    }

    .nestable-item:first-child,
    .nestable-item-copy:first-child {
      margin-top: 0;
    }

    .nestable-item .nestable-list,
    .nestable-item-copy .nestable-list {
      margin-top: 10px;
    }

    .nestable-item {
      position: relative;
    }
  }

  .handle {
    width: 100%;
    padding: 0 10px 0 0;
    height: 45px;
    line-height: 45px;
  }

  .nestable-item.is-dragging .nestable-list {
    pointer-events: none;
  }

  .nestable-item.is-dragging * {
    opacity: 0;
    filter: alpha(opacity=0);
  }

  .nestable-item.is-dragging:before {
    content: ' ';
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(106, 127, 233, 0.274);
    border: 1px dashed rgb(73, 100, 241);
    -webkit-border-radius: 5px;
    border-radius: 5px;
  }

  .nestable-drag-layer {
    position: fixed;
    top: 0;
    left: 0;
    z-index: 100;
    pointer-events: none;
  }

  .nestable-drag-layer > .nestable-list {
    position: absolute;
    top: 0;
    left: 0;
    padding: 0;
    background-color: rgba(106, 127, 233, 0.274);
  }

  .nestable [draggable='true'] {
    cursor: move;
  }

  .disabled {
    opacity: 0.5;

    &.cursor-pointer {
      cursor: not-allowed;
    }
  }

  .btn-cascade-open {
    transform: rotate(180deg);
    transform-origin: center center;
  }

  .hide-cascade > ol {
    display: none;
  }
}
</style>
