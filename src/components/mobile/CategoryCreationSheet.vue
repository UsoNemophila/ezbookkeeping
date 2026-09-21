<template>
    <f7-sheet swipe-to-close swipe-handler=".swipe-handler" style="height: auto"
              :opened="show" @sheet:open="onSheetOpen" @sheet:closed="onSheetClosed">
        <f7-toolbar class="toolbar-with-swipe-handler">
            <div class="swipe-handler"></div>
            <div class="left">
                <f7-link sheet-close icon-f7="xmark" :aria-label="tt('Close')"></f7-link>
            </div>
            <div class="right">
                <f7-link :text="tt('Save')"
                         :class="{ 'disabled': submitting || (mode === 'add' && !categoryName) }"
                         @click="save"></f7-link>
            </div>
        </f7-toolbar>
        <f7-page-content class="no-padding-top">
            <f7-block-title class="margin-top margin-horizontal">{{ sheetTitle }}</f7-block-title>

            <f7-list strong inset dividers class="margin-top" v-if="mode === 'add'">
                <f7-list-input
                    type="text"
                    clear-button
                    :label="tt('Category Name')"
                    :placeholder="tt('Your category name')"
                    :value="categoryName"
                    @input="categoryName = ($event.target as HTMLInputElement).value"
                ></f7-list-input>
            </f7-list>

            <f7-block class="no-padding no-margin category-creation-preset-list" v-if="mode === 'preset'">
                <f7-list strong inset dividers class="margin-top">
                    <f7-list-item :title="category.name"
                                  :key="idx"
                                  v-for="(category, idx) in flattenedPresetCategories">
                        <template #media>
                            <ItemIcon :icon-type="getCategoryIconType(category.iconType)"
                                      :icon-id="category.icon"
                                      :color="category.color"></ItemIcon>
                        </template>
                    </f7-list-item>
                </f7-list>
            </f7-block>
        </f7-page-content>
    </f7-sheet>
</template>

<script setup lang="ts">
import ItemIcon from '@/components/mobile/ItemIcon.vue';

import { ref, computed } from 'vue';

import { useI18n } from '@/locales/helpers.ts';
import { useI18nUIComponents, showLoading, hideLoading } from '@/lib/ui/mobile.ts';

import { useTransactionCategoriesStore } from '@/stores/transactionCategory.ts';

import { type LocalizedPresetCategory, CategoryType } from '@/core/category.ts';
import { TransactionCategory } from '@/models/transaction_category.ts';
import { categorizedArrayToPlainArray } from '@/lib/common.ts';
import { getCategoryIconType } from '@/lib/icon.ts';
import { localizedPresetCategoriesToTransactionCategoryCreateWithSubCategories } from '@/lib/category.ts';
import { generateRandomUUID } from '@/lib/misc.ts';

const props = defineProps<{
    show: boolean;
    // 'add' creates a single secondary category under parentId, 'preset' imports the whole default set
    mode: 'add' | 'preset';
    categoryType: CategoryType;
    parentId?: string;
    parentIcon?: string;
    parentColor?: string;
}>();

const emit = defineEmits<{
    (e: 'update:show', value: boolean): void;
    (e: 'category:saved', event: { message: string }): void;
}>();

const { tt, getCurrentLanguageTag, getAllTransactionDefaultCategories } = useI18n();
const { showToast } = useI18nUIComponents();

const transactionCategoriesStore = useTransactionCategoriesStore();

const categoryName = ref<string>('');
const submitting = ref<boolean>(false);

const sheetTitle = computed<string>(() => props.mode === 'add' ? tt('Add Secondary Category') : tt('Preset Categories'));

const allPresetCategories = computed<Record<string, LocalizedPresetCategory[]>>(
    () => getAllTransactionDefaultCategories(props.categoryType, getCurrentLanguageTag()));

const flattenedPresetCategories = computed<LocalizedPresetCategory[]>(
    () => categorizedArrayToPlainArray(allPresetCategories.value) as LocalizedPresetCategory[]);

function close(): void {
    emit('update:show', false);
}

function save(): void {
    if (props.mode === 'add') {
        saveNewCategory();
    } else {
        savePresetCategories();
    }
}

function saveNewCategory(): void {
    if (!categoryName.value || !props.parentId) {
        return;
    }

    const category = TransactionCategory.createNewCategory(props.categoryType, props.parentId);
    category.name = categoryName.value;

    if (props.parentIcon) {
        category.icon = props.parentIcon;
    }

    if (props.parentColor) {
        category.color = props.parentColor;
    }

    submitting.value = true;
    showLoading(() => submitting.value);

    transactionCategoriesStore.saveCategory({
        category: category,
        isEdit: false,
        clientSessionId: generateRandomUUID()
    }).then(() => {
        submitting.value = false;
        hideLoading();
        emit('category:saved', { message: 'You have added a new category' });
        close();
    }).catch(error => {
        submitting.value = false;
        hideLoading();

        if (!error.processed) {
            showToast(error.message || error);
        }
    });
}

function savePresetCategories(): void {
    const presetCategories = categorizedArrayToPlainArray(allPresetCategories.value);
    const submitCategories = localizedPresetCategoriesToTransactionCategoryCreateWithSubCategories(presetCategories);

    submitting.value = true;
    showLoading(() => submitting.value);

    transactionCategoriesStore.addPresetCategories({
        categories: submitCategories
    }).then(() => {
        submitting.value = false;
        hideLoading();
        emit('category:saved', { message: 'You have added preset categories' });
        close();
    }).catch(error => {
        submitting.value = false;
        hideLoading();

        if (!error.processed) {
            showToast(error.message || error);
        }
    });
}

function onSheetOpen(): void {
    categoryName.value = '';
    submitting.value = false;
}

function onSheetClosed(): void {
    emit('update:show', false);
}
</script>

<style>
.category-creation-preset-list {
    max-height: 50vh;
    overflow-y: auto;
}
</style>
