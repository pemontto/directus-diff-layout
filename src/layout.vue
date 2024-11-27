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
    import { useStores, useSync } from "@directus/extensions-sdk";

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

    const { t } = useI18n();
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
    </div>
</template>

<style lang="scss" scoped>
    .custom-layout {
        display: contents;
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
</style>
