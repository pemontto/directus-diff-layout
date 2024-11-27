<script setup lang="ts">
    import {
        inject,
        ref,
        toRefs,
        watch,
        computed,
        type ComponentPublicInstance,
        type Ref,
    } from "vue";
    import { useI18n } from "vue-i18n";
    import type { ShowSelect } from "@directus/extensions";
    import type { Field, Filter, Item } from "@directus/types";
    // CORE CLONES
    import { HeaderRaw } from "./core-clones/components/v-table/types";
    import {
        AliasFields,
        useAliasFields,
    } from "./core-clones/composables/use-alias-fields";
    import { usePageSize } from "./core-clones/composables/use-page-size";
    import { useShortcut } from "./core-clones/composables/use-shortcut";
    import { Collection } from "./core-clones/types/collections";
    // CORE CHANGES
    // import { useSync } from '@directus/composables';
    // import { useCollectionPermissions } from '@/composables/use-permissions';
    import { useStores, useSync, useItems, useApi } from "@directus/extensions-sdk";

    defineOptions({ inheritAttrs: false });

    interface Props {
        collection: string;
        selection?: Item[];
        readonly: boolean;
        tableHeaders: HeaderRaw[];
        showSelect?: ShowSelect;
        items: Item[];
        loading: boolean;
        error?: any;
        totalPages: number;
        tableSort?: { by: string; desc: boolean } | null;
        onRowClick: ({
            item,
            event,
        }: {
            item: Item;
            event: PointerEvent;
        }) => void;
        tableRowHeight: number;
        page: number;
        toPage: (newPage: number) => void;
        itemCount?: number;
        fields: string[];
        limit: number;
        primaryKeyField?: Field;
        info?: Collection;
        sortField?: string;
        changeManualSort: (data: any) => Promise<void>;
        resetPresetAndRefresh: () => Promise<void>;
        selectAll: () => void;
        filterUser?: Filter;
        search?: string;
        aliasedFields: Record<string, AliasFields>;
        aliasedKeys: string[];
        onSortChange: (newSort: { by: string; desc: boolean }) => void;
        onAlignChange?: (
            field: "string",
            align: "left" | "center" | "right"
        ) => void;
    }

    const props = withDefaults(defineProps<Props>(), {
        selection: () => [],
        showSelect: "none",
        error: null,
        itemCount: undefined,
        tableSort: undefined,
        primaryKeyField: undefined,
        info: undefined,
        sortField: undefined,
        filterUser: undefined,
        search: undefined,
        onAlignChange: () => undefined,
    });

    const emit = defineEmits([
        "update:selection",
        "update:tableHeaders",
        "update:limit",
        "update:fields",
    ]);

    const { t } = useI18n(); // Ensure useI18n is correctly initialized
    const { collection } = toRefs(props);

    // CORE CHANGES
    const { sortAllowed } = useCollectionPermissions(collection);
    function useCollectionPermissions(collection: Ref<string>) {
        const { usePermissionsStore } = useStores();
        const permissionsStore = usePermissionsStore();

        return {
            sortAllowed: computed(() => {
                if (!props.sortField) return false;
                return permissionsStore.hasPermission(collection.value, "sort");
            }),
        };
    }

    const selectionWritable = useSync(props, "selection", emit);
    const tableHeadersWritable = useSync(props, "tableHeaders", emit);
    const limitWritable = useSync(props, "limit", emit);

    const mainElement = inject<Ref<Element | undefined>>("main-element");

    const table = ref<ComponentPublicInstance>();

    watch(
        () => props.page,
        () => mainElement?.value?.scrollTo({ top: 0, behavior: "smooth" })
    );

    useShortcut(
        "meta+a",
        () => {
            props.selectAll();
        },
        table
    );

    const { sizes: pageSizes, selected: selectedSize } = usePageSize<string>(
        [25, 50, 100, 250, 500, 1000],
        (value) => String(value),
        props.limit
    );

    if (limitWritable.value !== selectedSize) {
        limitWritable.value = selectedSize;
    }

    const fieldsWritable = useSync(props, "fields", emit);

    const { getFromAliasedItem } = useAliasFields(fieldsWritable, collection);

    function addField(fieldKey: string) {
        fieldsWritable.value = [...fieldsWritable.value, fieldKey];
    }

    function removeField(fieldKey: string) {
        fieldsWritable.value = fieldsWritable.value.filter(
            (field) => field !== fieldKey
        );
    }

    // console.log(loading)
    // console.log(items)

    const selectedItem1 = ref(null);
    const selectedItem2 = ref(null);
    const itemDetails1 = ref<Item | null>(null);
    const itemDetails2 = ref<Item | null>(null);

    watch(selectedItem1, async (newItemId) => {
        if (newItemId) {
            console.log("Item details for selectedItem1:", newItemId);
            console.log(`Calling useItems for id:'${newItemId}' in collection:'${collection.value}'`)
            try {
                itemDetails1.value = await useItems(collection.value, newItemId);
                console.log("Fetched item details for selectedItem1:", itemDetails1.value);
            } catch (error) {
                console.error("Error fetching item details:", error);
            }
        } else {
            itemDetails1.value = null;
        }
    });


    watch(selectedItem2, async (newItemId) => {
        if (newItemId) {
            console.log("Item details for selectedItem2:", newItemId);
            console.log(`Calling useItems for id:'${newItemId}' in collection:'${collection.value}'`)
            try {
                itemDetails2.value = await useItems(collection.value, newItemId);
                console.log("Fetched item details for selectedItem2:", itemDetails2.value);
            } catch (error) {
                console.error("Error fetching item details:", error);
            }
        } else {
            itemDetails2.value = null;
        }
    });
</script>

<template>
    <div class="custom-layout">
        <slot
            v-if="error"
            name="error"
            :error="error"
            :reset="resetPresetAndRefresh"
        />
        <slot
            v-else-if="itemCount === 0 && (filterUser || search)"
            name="no-results"
        />
        <slot
            v-else-if="itemCount === 0"
            name="no-items"
        />
        <div class="split-pane">
            <div class="column">
                <v-select
                    :items="items"
                    label="Select Item"
                    v-model="selectedItem1"
                    item-text="id"
                    item-value="id"
                />
                <!-- Content for the first column goes here -->
                <div class="content">
                    <!-- Placeholder for the first column content -->
                    <p>Content for the first column</p>
                </div>
            </div>
            <div class="column">
                <!-- Add dropdown for selecting an item from the filtered Directus collection -->
                <v-select
                    :items="items"
                    label="Select Item"
                    v-model="selectedItem2"
                    item-text="id"
                    item-value="id"
                />
                <!-- Content for the second column goes here -->
                <div class="content">
                    <!-- Placeholder for the second column content -->
                    <p>Content for the second column</p>
                </div>
            </div>
        </div>
    </div>
</template>

<style lang="scss" scoped>
.custom-layout {
        display: flex;
        flex-direction: column;
        margin: var(--content-padding);
        margin-bottom: var(--content-padding-bottom);
    }

    .v-table {
        --v-table-sticky-offset-top: var(--layout-offset-top);

        display: contents;

        & > :deep(table) {
            min-width: calc(100% - var(--content-padding)) !important;
            margin-left: var(--content-padding);

            tr {
                margin-right: var(--content-padding);
            }
        }
    }

    .footer {
        position: sticky;
        left: 0;
        display: flex;
        align-items: center;
        justify-content: space-between;
        width: 100%;
        padding: 32px var(--content-padding);

        .pagination {
            display: inline-block;
        }

        .per-page {
            display: flex;
            align-items: center;
            justify-content: flex-end;
            width: 240px;
            color: var(--theme--foreground-subdued);

            span {
                width: auto;
                margin-right: 4px;
            }

            .v-select {
                color: var(--theme--foreground);
            }
        }
    }

    .add-field {
        --v-icon-color-hover: var(--theme--foreground);

        &.active {
            --v-icon-color: var(--theme--foreground);
        }
    }

    .flip {
        transform: scaleY(-1);
    }
.split-pane {
        display: flex;
        flex-direction: row;
        gap: 16px; /* Adjust the gap between columns as needed */
    }

.column {
        flex: 1;
        display: flex;
        flex-direction: column;
    }

.content {
        flex: 1;
        padding: 16px; /* Adjust the padding as needed */
        border: 1px solid var(--theme--border);
        border-radius: 4px;
        background-color: var(--theme--background);
    }
</style>
