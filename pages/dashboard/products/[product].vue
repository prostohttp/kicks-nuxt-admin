<script lang="ts" setup>
import type { FormErrorEvent } from "#ui/types";
import { locale } from "~/lang/locale";
import { validate } from "./validator";
import type { BreadcrumbItem } from "~/types/ui/ui.types";
import type { FlatProductDto } from "~/server/api/product/dto/product.dto";

// store
const productDataStore = useProductDataStore();
const brandDataStore = useBrandDataStore();
const categoryDataStore = useCategoryDataStore();
const settingsDataStore = useSettingsDataStore();
const optionDataStore = useOptionDataStore();

await useAsyncData(() =>
    productDataStore.getProductById(
        useRoute("dashboard-products-product").params.product,
    ),
);
const { product } = storeToRefs(productDataStore);
await useAsyncData(() => categoryDataStore.getAllTitles());
await useAsyncData(() => optionDataStore.getAllOptionsWithoutPagination());

const { brand } = storeToRefs(brandDataStore);
const { titles: categoryTitles } = storeToRefs(categoryDataStore);
const { locale: storeLocale } = storeToRefs(settingsDataStore);

// vars
const toast = useToast();
const router = useRouter();
const isAdmin = checkIsAdmin();
const route = useRoute();
const fullPath = router.currentRoute.value.fullPath;
const isValidForm = ref(true);
const links: Ref<BreadcrumbItem[]> = ref(
    breadcrumbsArrayFactory(fullPath, product.value?.title, fullPath),
);
const title = computed(() =>
    product.value
        ? product.value?.title
        : locale[settingsDataStore.locale].empty,
);
const items = [
    {
        slot: "data",
        label: locale[storeLocale.value].productData,
        tab: "data",
    },
    {
        slot: "options",
        label: locale[storeLocale.value].productOptions,
        tab: "options",
    },
];

const state: Ref<FlatProductDto> = ref({
    ...product.value!,
    category: product.value?.category!.map((item) => item.title)!,
    brand: product.value?.brand?.title!,
    options: product.value?.options!,
});

// handlers
const onSubmitHandler = async () => {
    try {
        if (state.value.brand) {
            await brandDataStore.getBrandByTitle(state.value.brand);
        }
        const data = {
            ...state.value,
            options: state.value?.options.map((option) => {
                const values = [];
                for (const opt of option.values!) {
                    if (opt.optionValue.value.label) {
                        values.push({
                            ...opt,
                            optionValue: opt.optionValue.value.value,
                        });
                    } else if (opt.optionValue.value.label === "") {
                        continue;
                    } else {
                        values.push({
                            ...opt,
                            optionValue: opt.optionValue._id,
                        });
                    }
                }
                return {
                    ...option,
                    optionValue: option.optionValue._id,
                    values: values,
                };
            }),
            category: categoryTitles.value
                .filter((el) => state.value.category?.includes(el.title))
                .map((el) => el._id),
            brand: brand.value?._id || null,
        };

        await $fetch("/api/product/edit", {
            method: "PUT",
            body: data,
        });
        isValidForm.value = true;
        await productDataStore.getProductCount();
        await productDataStore.getProductById(state.value._id!);
        toast.add({
            title: locale[storeLocale.value].successEdit,
            color: "green",
        });
    } catch (error: any) {
        toast.add({ title: error.message, color: "red" });
    }
};

const selected = computed({
    get() {
        const index = items.findIndex((item) => item.tab === route.query.tab);
        if (index === -1) {
            return 0;
        }
        return index;
    },
    set(val) {
        router.replace({
            query: { tab: items[val].tab },
        });
    },
});

const onSubmit = useThrottleFn(onSubmitHandler, 3000);

const protectedSubmitHandler = computed(() => (isAdmin ? onSubmit : () => {}));

async function onError(event: FormErrorEvent) {
    if (
        event.errors.length ||
        (product.value?.options && product.value.options.length > 1)
    ) {
        isValidForm.value = false;
    } else {
        isValidForm.value = true;
    }
}

// hooks
watch(product, () => {
    links.value = breadcrumbsArrayFactory(
        fullPath,
        product.value?.title,
        fullPath,
    );
});

// meta
useHead({
    title: title,
});
</script>

<template>
    <DashboardBreadcrumbs
        :links="links"
        :title="
            product ? product.title : locale[settingsDataStore.locale].empty
        "
    />
    <main
        class="p-[24px] bg-white rounded-[16px] dark:bg-dark-gray dark:border border-[#70706e]"
    >
        <div class="flex lg:flex-row flex-col lg:gap-[35px] gap-[20px]">
            <LazyUiEmpty v-if="!product" />
            <div v-else class="space-y-4 w-full">
                <div
                    v-if="!isValidForm"
                    class="bg-dark-gray dark:bg-yellow text-fa-white dark:text-dark-gray w-full text-center py-[5px] mb-[10px] rounded-[8px]"
                >
                    {{
                        locale[settingsDataStore.locale].error
                            .checkRequiredFields
                    }}
                </div>
                <UForm
                    :state="state"
                    :validate="validate"
                    class="flex flex-col lg:gap-[35px] gap-[20px]"
                    @error="onError"
                    @submit="protectedSubmitHandler"
                >
                    <UTabs v-model="selected" :items="items" class="w-full">
                        <template #data>
                            <DashboardProductDataTab
                                v-model:state="state"
                                :isSaved="true"
                            />
                        </template>
                        <template #options>
                            <DashboardProductOptionTab
                                v-model:options="state.options"
                            />
                        </template>
                    </UTabs>
                    <div>
                        <UButton
                            v-if="isAdmin"
                            class="dark-button my-[10px]"
                            type="submit"
                        >
                            {{ locale[storeLocale].save }}
                        </UButton>
                    </div>
                </UForm>
            </div>
        </div>
        <!-- <pre>{{ state }}</pre> -->
    </main>
</template>
