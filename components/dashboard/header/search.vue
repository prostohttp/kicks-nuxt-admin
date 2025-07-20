<script lang="ts" setup>
import { onClickOutside, useDebounceFn, type MaybeElementRef } from "@vueuse/core";
import { locale } from "~/lang/locale";
import { ModelNamesForSearchEngine } from "~/types/server/server.types";
import type { SearchProductDto } from "./dto/search-product.dto";
import { Constants } from "~/constants";

// vars
const query = ref("");
const target = useTemplateRef("target");
const isActive = ref(false);
const isOpen = ref(false);
const founded: Ref<SearchProductDto[] | undefined> = ref();
const metaPressed = ref(false);

// store
const settingsDataStore = useSettingsDataStore();

// handlers
onClickOutside(target as MaybeElementRef, () => (isOpen.value = false));

const showSearchResultHandler = () => {
    if (!query.value) {
        isOpen.value = false;
    } else {
        isOpen.value = true;
    }
};

const searchHandler = async () => {
    const data = await $fetch("/api/search/all", {
        method: "GET",
        query: {
            searchModel: ModelNamesForSearchEngine.PRODUCT,
            searchPhrase: query.value,
            limit: Constants.SEARCH_LIMIT,
        },
        pick: ["results"],
    });
    founded.value = data.result.products.data;
};

const debounceSearchHandler = useDebounceFn(
    () => {
        searchHandler();
    },
    1000,
    {
        maxWait: 5000,
    },
);

const prettySearchHandler = (event: KeyboardEvent) => {
    if (event.code === "ArrowDown" || event.code === "Enter") {
        if (query.value) {
            isOpen.value = true;
        }
    }
    if (event.code === "Escape") {
        isOpen.value = false;
    }
};

const showSearchInputHandler = () => {
    isActive.value = true;
    if (query.value) {
        isOpen.value = true;
    }
};

const hideSearchInputHandler = () => {
    isOpen.value = false;
    isActive.value = false;
};

const searchHandlerWithShortcut = (event: KeyboardEvent) => {
    if (event.code === "ControlLeft") {
        metaPressed.value = true;
    }
    if (metaPressed.value && event.code === "Enter" && query.value) {
        hideSearchInputHandler();
        metaPressed.value = false;
        return navigateTo(`/dashboard/search?searchPhrase=${query.value}`);
    }
};
// hooks
watch(query, (oldValue, newValue) => {
    showSearchResultHandler();
    if (oldValue !== newValue) {
        debounceSearchHandler();
    }
});

// TODO: locale[settingsDataStore.locale].search конструкция вызывает запрос к бд!
</script>

<template>
    <div class="flex items-center gap-[10px]">
        <div class="flex justify-center items-center w-[40px] h-[40px]">
            <UIcon
                v-if="!isActive"
                class="bg-dark-gray hover:bg-blue dark:bg-fa-white dark:hover:bg-yellow w-[24px] h-[24px] cursor-pointer"
                name="i-heroicons-magnifying-glass-20-solid"
                @click="showSearchInputHandler"
            />
        </div>
        <UInput
            v-if="isActive"
            ref="target"
            v-model="query"
            :placeholder="`${locale[settingsDataStore.locale].search}...`"
            :ui="{
                icon: {
                    trailing: { pointer: '' },
                },
            }"
            autocomplete="off"
            autofocus
            icon="i-heroicons-magnifying-glass-20-solid"
            input-class=" border-dark-gray dark:border-fa-white h-[40px] pl-[44px] text-dark-gray sm:w-[200px] w-[calc(100%-30px)] z-[100] sm:relative sm:left-0 sm:top-0 fixed left-[20px] top-[32px]"
            name="query"
            @keydown="searchHandlerWithShortcut"
            @keyup="prettySearchHandler"
        >
            <Transition>
                <div
                    v-if="isOpen"
                    class="top-[90px] right-[10px] md:right-0 z-[101] fixed md:absolute flex flex-col gap-[16px] bg-white dark:bg-dark-bg p-[20px] rounded-[8px] w-[calc(100%-30px)] md:w-auto md:min-w-[250px] text-dark-gray dark:text-fa-white"
                >
                    <h3 class="font-[600] font-[Rubik] text-[20px]">
                        {{ locale[settingsDataStore.locale].searchResult }}
                    </h3>
                    <DashboardHeaderSearchList
                        v-model="isOpen"
                        :data="founded"
                    />
                    <NuxtLink
                        v-if="founded && founded.length"
                        :to="`/dashboard/search?searchPhrase=${query}`"
                        class="font-[600] text-[16px] text-blue hover:text-dark-gray dark:hover:text-fa-white dark:text-yellow"
                        @click="hideSearchInputHandler"
                    >
                        {{ locale[settingsDataStore.locale].seeAll }}
                    </NuxtLink>
                </div>
            </Transition>
            <template #leading>
                <UButton
                    :padded="true"
                    class="top-[37px] left-[25px] z-[101] sm:static fixed text-dark-gray hover:text-blue dark:text-fa-white"
                    icon="i-heroicons-magnifying-glass-20-solid"
                    variant="link"
                />
            </template>
            <template #trailing>
                <UButton
                    :padded="true"
                    class="top-[37px] right-[20px] z-[101] sm:static fixed text-dark-gray hover:text-blue dark:text-fa-white"
                    icon="i-heroicons-x-mark-20-solid"
                    variant="link"
                    @click="hideSearchInputHandler"
                />
            </template>
        </UInput>
    </div>
</template>
