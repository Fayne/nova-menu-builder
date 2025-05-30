<template>
  <Modal :show="showModal" class="bg-white dark:bg-gray-800 rounded-lg shadow-lg overflow-hidden">
    <ModalHeader v-text="__('novaMenuBuilder.copyMenuItemsFromTitle')" />

    <form @submit.prevent="copyMenuItems" autocomplete="off">
      <DefaultField
        :field="{
          fullWidth: true,
          stacked: true,
          withLabel: true,
          visible: true,
          name: __('novaMenuBuilder.menuResourceSingularLabel'),
        }"
      >
        <template #field>
          <SelectControl
            v-if="hasMultipleMenus"
            :options="menuOptions.map(v => ({ value: v.id, label: v.name }))"
            :placeholder="__('novaMenuBuilder.menuResourceSingularLabel')"
            v-model="selectedMenu"
            @change="selectedMenu = $event"
          />
        </template>
      </DefaultField>

      <DefaultField
        :field="{
          fullWidth: true,
          stacked: true,
          withLabel: true,
          visible: true,
          name: __('novaMenuBuilder.locale'),
        }"
      >
        <template #field>
          <SelectControl
            :options="localeOptions.map(v => ({ value: v.id, label: v.name }))"
            :placeholder="__('novaMenuBuilder.locale')"
            v-model="selectedLocale"
            @change="selectedLocale = $event"
          />
        </template>
      </DefaultField>
    </form>

    <ModalFooter class="flex justify-end">
      <div class="ml-auto">
        <Button
          state="danger"
          dusk="cancel-action-button"
          @click.prevent="$emit('closeModal')"
          :label="__('novaMenuBuilder.closeModalTitle')"
          class="mr-3"
        />

        <Button
          type="button"
          dusk="confirm-action-button"
          state="default"
          variant="solid"
          :disabled="isCopying"
          :loading="isCopying"
          :label="__('novaMenuBuilder.copyMenuItemsButtonTitle')"
          @click.prevent="copyMenuItemsFromMenu"
        />
      </div>
    </ModalFooter>
  </Modal>
</template>

<script>
import api from '../../api';
import { Button } from 'laravel-nova-ui';

export default {
  props: ['showModal', 'activeLocale', 'locales', 'resourceName', 'resourceId', 'menuCount'],
  data: () => ({
    isCopying: false,
    menuOptions: [],
    selectedMenu: void 0,
    selectedLocale: void 0,
    localeOptions: [],
  }),

  components: { Button },

  async mounted() {
    this.setLocaleDataBasedOnActiveLocale();
    await this.fetchMenuOptions();
  },

  watch: {
    activeLocale() {
      this.setLocaleDataBasedOnActiveLocale();
    },
    resourceId() {
      this.setLocaleDataBasedOnActiveLocale();
    },
    selectedMenu() {
      this.setLocaleDataBasedOnActiveLocale();
    },
  },

  methods: {
    async fetchMenuOptions() {
      let menuOptions = (await api.getMenus({ notEmpty: true })).data;

      if (this.hasMultipleLocales) {
        this.selectedMenu = String(this.resourceId);
      } else {
        // Just one locale, let's remove the current menu from the selection
        menuOptions = menuOptions.filter(m => String(m.id) !== String(this.resourceId));
      }

      this.menuOptions = menuOptions;
      this.selectedMenu = menuOptions[0] ? menuOptions[0].id : void 0;
    },

    async copyMenuItemsFromMenu() {
      if (!this.selectedMenu || !this.selectedLocale) return;
      this.isCopying = true;
      try {
        await api.copyItems(this.selectedMenu, this.selectedLocale, this.resourceId, this.activeLocale);
        this.$emit('refreshItems');
        this.$emit('closeModal');
      } catch (e) {
        console.info(e);
      }
      this.isCopying = false;
    },

    setLocaleDataBasedOnActiveLocale() {
      let options = Object.keys(this.locales).map(key => ({
        id: key,
        name: this.locales[key],
      }));

      // If it's the same menu, don't show current locale
      if (this.selectedMenu && String(this.resourceId) === String(this.selectedMenu)) {
        options = options.filter(obj => obj.id !== this.activeLocale);
      }

      this.localeOptions = options;
      this.selectedLocale = options[0].id;
    },
  },
  computed: {
    hasMultipleLocales() {
      return Object.keys(this.locales).length > 1;
    },

    hasMultipleMenus() {
      return this.menuCount > 1;
    },
  },
};
</script>
