<script lang="ts" setup>
import { useThrottleFn } from "@vueuse/core";
import { locale } from "~/lang/locale";
import {
    type EmailRegisterDto,
    type RegisterFormDto,
    Roles,
} from "~/types/server/server.types";
import { Constants } from "~/constants";

// store
const settingsDataStore = useSettingsDataStore();

// vars
const toast = useToast();
const { signIn } = useAuth();

// meta
definePageMeta({
    layout: "auth",
    name: "register",
    auth: {
        unauthenticatedOnly: true,
        navigateAuthenticatedTo: "/dashboard",
    },
    alias: ["/register"],
});
useHead({
    title: locale[useSettingsDataStore().locale].register,
});

// Handlers
const register = async (data: RegisterFormDto) => {
    const { name, email, password, keepLogged } = data;
    try {
        await $fetch("/api/auth/register", {
            method: "POST",
            body: {
                name,
                email,
                password,
                role: Roles.MANAGER,
                keepLogged,
                image: "",
            },
        });

        toast.add({
            title: "Registered, please wait",
        });

        await $fetch("/api/email/register-send-email", {
            method: "POST",
            body: {
                name,
                userEmail: email,
                siteName: locale[useSettingsDataStore().locale].siteName,
                siteUrl: Constants.SITE_URL + "/login",
            } as EmailRegisterDto,
        });

        if (!data.keepLogged) {
            toast.add({
                title: "You are have been registered and now you can login",
                callback: () => {
                    return navigateTo("/login");
                },
            });
        } else {
            await signIn("credentials", {
                email,
                password,
            });
        }
    } catch (error: any) {
        toast.add({ title: error.statusMessage });
    }
};
const registerHandler = useThrottleFn(register, 1000);
</script>

<template>
    <div class="flex flex-col lg:grid lg:grid-cols-2 h-dvh dark:text-white">
        <AuthSidebar />
        <div
            class="flex justify-center items-center px-[30px] lg:px-0 py-[10px]"
        >
            <div class="flex flex-col gap-[24px] w-full max-w-[480px]">
                <div>
                    <h1 class="font-[600] font-[Rubik] text-[36px]">
                        {{ locale[settingsDataStore.locale].signUp }}
                    </h1>
                    <h2 class="font-[600] text-[20px] open-sans"></h2>
                </div>
                <AuthSocialButtons />
                <h2 class="font-[600] font-[Rubik] text-[36px]">
                    {{ locale[settingsDataStore.locale].register }}
                </h2>
                <AuthRegisterForm @submit="registerHandler" />
            </div>
        </div>
    </div>
</template>
