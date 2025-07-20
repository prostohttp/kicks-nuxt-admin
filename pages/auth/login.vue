<script lang="ts" setup>
import { useThrottleFn } from "@vueuse/core";
import { locale } from "~/lang/locale";
import { type LoginFormDto } from "~/types/server/server.types";

// store
const settingsDataStore = useSettingsDataStore();

// Vars
const isOpen = ref(false);
const isError = ref();
const { signIn } = useAuth();
const toast = useToast();

// meta
definePageMeta({
    layout: "auth",
    name: "login",
    auth: {
        unauthenticatedOnly: true,
        navigateAuthenticatedTo: "/dashboard",
    },
    alias: ["/login"],
});

useHead({
    title: locale[settingsDataStore.locale].login,
});

// Handlers
const login = async (data: LoginFormDto) => {
    isError.value = await signIn("credentials", {
        email: data.email,
        password: data.password,
        redirect: false,
    });
    if (!isError.value.error) {
        return navigateTo("/dashboard");
    } else {
        toast.add({ title: locale[settingsDataStore.locale].error.wrongData });
    }
};
const loginHandler = useThrottleFn(login, 1000);
</script>

<template>
    <div class="flex flex-col lg:grid lg:grid-cols-2 h-dvh dark:text-white">
        <AuthSidebar />
        <div
            class="flex justify-center items-center px-[30px] lg:px-0 pt-[40px] lg:pt-[40px] pb-[40px]"
        >
            <div class="flex flex-col gap-[24px] w-full max-w-[480px]">
                <div>
                    <h1 class="mb-[8px] font-[600] font-[Rubik] text-[36px]">
                        {{ locale[settingsDataStore.locale].login }}
                    </h1>
                    <ULink
                        class="font-[600] text-[16px] decoration-gray-main underline open-sans"
                        to="/auth/forgot"
                    >
                        {{ locale[settingsDataStore.locale].forgotPassword }}
                    </ULink>
                </div>
                <AuthLoginForm @submit="loginHandler" />
                <AuthSocialButtons />
                <span
                    class="font-[600] text-[16px] decoration-gray-main underline cursor-pointer open-sans hover:"
                    @click.prevent="isOpen = true"
                >
                    {{ locale[settingsDataStore.locale].terms }}
                </span>
            </div>
        </div>
        <UiModal
            v-model="isOpen"
            class="dark:text-white"
            fullscreen
            title="KicksClub Terms & Conditions"
        >
            <ContentDoc class="dark:text-white" path="/terms" />
        </UiModal>
    </div>
</template>
