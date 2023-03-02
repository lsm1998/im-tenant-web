<template>
    <div class="login">
        <el-container class="is-vertical">
            <Header/>
            <el-main class="frontend-footer-brother">
                <el-row justify="center">
                    <el-col :span="16" :xs="24">
                        <div v-if="true" class="login-box">
                            <div class="login-title">
                                租户登录
                            </div>
                            <el-form ref="formRef" @keyup.enter="onSubmit(formRef)" :rules="rules" :model="state.form">
                                <!-- 登录注册用户名 -->
                                <el-form-item prop="username">
                                    <el-input
                                        v-model="state.form.username"
                                        placeholder="请输入用户名"
                                        :clearable="true"
                                        size="large"
                                    >
                                        <template #prefix>
                                            <Icon name="fa fa-user" size="16" color="var(--el-input-icon-color)"/>
                                        </template>
                                    </el-input>
                                </el-form-item>

                                <!-- 登录注册密码 -->
                                <el-form-item prop="password">
                                    <el-input
                                        v-model="state.form.password"
                                        placeholder="请输入密码"
                                        type="password"
                                        show-password
                                        size="large"
                                    >
                                        <template #prefix>
                                            <Icon name="fa fa-unlock-alt" size="16" color="var(--el-input-icon-color)"/>
                                        </template>
                                    </el-input>
                                </el-form-item>

                                <!-- 登录验证码 -->
                                <el-form-item v-if="state.form.tab == 'login'" prop="captcha">
                                    <el-row class="w100">
                                        <el-col :span="16">
                                            <el-input
                                                v-model="state.form.captcha"
                                                clearable
                                                autocomplete="off"
                                                placeholder="请输入验证码"
                                                size="large"
                                            >
                                                <template #prefix>
                                                    <Icon name="fa fa-ellipsis-h" size="16"
                                                          color="var(--el-input-icon-color)"/>
                                                </template>
                                            </el-input>
                                        </el-col>
                                        <el-col class="captcha-box" :span="8">
                                            <img
                                                @click="onChangeCaptcha"
                                                class="captcha-img"
                                                :src="buildCaptchaUrl() + '&id=' + state.form.captchaId"
                                                alt=""
                                            />
                                        </el-col>
                                    </el-row>
                                </el-form-item>

                                <div class="form-footer">
                                    <el-checkbox v-model="state.form.keep" label="记住我"
                                                 size="default"></el-checkbox>
                                    <div
                                        v-if="state.accountVerificationType.length > 0"
                                        @click="state.showRetrievePasswordDialog = true"
                                        class="forgot-password"
                                    >
                                        {{ t('user.login.Forgot your password?') }}
                                    </div>
                                </div>
                                <el-form-item class="form-buttons">
                                    <el-button
                                        class="login-btn"
                                        @click="onSubmit(formRef)"
                                        :loading="state.formLoading"
                                        round
                                        type="primary"
                                        size="large"
                                    >
                                        登录
                                    </el-button>
                                </el-form-item>
                                <LoginFooterMixin/>
                            </el-form>
                        </div>
                    </el-col>
                </el-row>
            </el-main>
            <Footer/>
        </el-container>
    </div>
</template>

<script setup lang="ts">
import {reactive, onMounted, onUnmounted, ref} from 'vue'
import Header from '/@/layouts/frontend/components/header.vue'
import Footer from '/@/layouts/frontend/components/footer.vue'
import {useSiteConfig} from '/@/stores/siteConfig'
import {useMemberCenter} from '/@/stores/memberCenter'
import {buildCaptchaUrl, sendEms, sendSms} from '/@/api/common'
import {uuid} from '/@/utils/random'
import {useI18n} from 'vue-i18n'
import {buildValidatorData, validatorAccount} from '/@/utils/validate'
import {checkIn, login, retrievePassword} from '/@/api/frontend/user/index'
import {useEventListener} from '@vueuse/core'
import {onResetForm} from '/@/utils/common'
import {useUserInfo} from '/@/stores/userInfo'
import {useRouter} from 'vue-router'
import {useRoute} from 'vue-router'
import loginMounted from '/@/components/mixins/loginMounted'
import LoginFooterMixin from '/@/components/mixins/loginFooter.vue'
import type {FormItemRule, FormInstance} from 'element-plus'
import {dayjs} from "element-plus";

let timer: NodeJS.Timer

const {t} = useI18n()
const route = useRoute()
const router = useRouter()
const userInfo = useUserInfo()
const siteConfig = useSiteConfig()
const memberCenter = useMemberCenter()
const formRef = ref<FormInstance>()

interface State {
    form: {
        tab: 'login' | 'register'
        email: string
        mobile: string
        username: string
        password: string
        captcha: string
        keep: boolean
        captchaId: string
        registerType: 'email' | 'mobile'
    }
    formLoading: boolean
    showCaptcha: boolean
    showRetrievePasswordDialog: boolean
    retrievePasswordForm: {
        type: 'email' | 'mobile'
        account: string
        captcha: string
        password: string
    }
    dialogWidth: number
    accountVerificationType: string[]
    codeSendCountdown: number
    submitRetrieveLoading: boolean
    sendCaptchaLoading: boolean
}

const state: State = reactive({
    form: {
        tab: 'login',
        email: '',
        mobile: '',
        username: '',
        password: '',
        captcha: '',
        keep: false,
        captchaId: uuid(),
        registerType: 'email',
    },
    formLoading: false,
    showCaptcha: false,
    showRetrievePasswordDialog: false,
    retrievePasswordForm: {
        type: 'email',
        account: '',
        captcha: '',
        password: '',
    },
    dialogWidth: 36,
    accountVerificationType: [],
    codeSendCountdown: 0,
    submitRetrieveLoading: false,
    sendCaptchaLoading: false,
})

const rules: Partial<Record<string, FormItemRule[]>> = reactive({
    email: [
        buildValidatorData({name: 'required', title: t('user.login.email')}),
        buildValidatorData({name: 'email', title: t('user.login.email')}),
    ],
    username: [
        buildValidatorData({name: 'required', title: "请输入用户名"}),
        {
            validator: (rule: any, val: string, callback: Function) => {
                if (state.form.tab == 'register') {
                    return validatorAccount(rule, val, callback)
                } else {
                    callback()
                }
            },
            trigger: 'blur',
        },
    ],
    password: [buildValidatorData({name: 'required', title: "请输入密码"}), buildValidatorData({name: 'password'})],
    mobile: [buildValidatorData({name: 'required', title: "请输入手机号"}), buildValidatorData({name: 'mobile'})],
    captcha: [buildValidatorData({name: 'required', title: "请输入验证码"})],
})

const retrieveRules: Partial<Record<string, FormItemRule[]>> = reactive({
    account: [buildValidatorData({name: 'required', title: "请输入用户名"})],
    captcha: [buildValidatorData({name: 'required', title: "请输入验证码"})],
    password: [buildValidatorData({
        name: 'required',
        title: "请输入密码"
    }), buildValidatorData({name: 'password'})],
})

const resize = () => {
    let clientWidth = document.documentElement.clientWidth
    let width = 36
    if (clientWidth <= 790) {
        width = 92
    } else if (clientWidth <= 910) {
        width = 56
    } else if (clientWidth <= 1260) {
        width = 46
    }
    state.dialogWidth = width
}

const onChangeCaptcha = () => {
    state.form.captcha = ''
    state.form.captchaId = uuid()
}
const onSubmit = (formRef: FormInstance | undefined = undefined) => {
    formRef!.validate((valid) => {
        if (valid) {
            state.formLoading = true
            login(state.form)
                .then((res) => {
                    userInfo.dataFill(res.data.userInfo)
                    router.push({path: '/user'})
                })
                .catch(() => {
                    onChangeCaptcha()
                    // todo 测试
                    router.push({path: '/user'})

                    userInfo.dataFill({
                        "id": 1,
                        "username": "user",
                        "nickname": "User",
                        "email": "user@buildadmin.com",
                        "mobile": "18888888889",
                        "avatar": "https://demo.buildadmin.com/storage/default/20220920/4f4d41d0860d8a1e9c07c199a40da39244f55d46.jpg",
                        "gender": 2,
                        "birthday": "2022-05-13",
                        "money": parseInt("10.00"),
                        "score": 10,
                        "lastlogintime": new Date(1677682849*1000).toString(),
                        "lastloginip": "113.116.131.103",
                        "jointime":  dayjs(1677682849*1000).format('YYYY-MM-DD'),
                        "motto": "",
                        "token": "cfc628bc-94c1-4a8a-b0b7-f501575df616",
                        "refreshToken": ""
                    })
                }).finally(() => {
                state.formLoading = false
            })
        } else {
            onChangeCaptcha()
        }
    })
}
const onSubmitRetrieve = (formRef: FormInstance | undefined = undefined) => {
    formRef!.validate((valid) => {
        if (valid) {
            state.submitRetrieveLoading = true
            retrievePassword(state.retrievePasswordForm)
                .then((res) => {
                    state.submitRetrieveLoading = false
                    if (res.code == 1) {
                        state.showRetrievePasswordDialog = false
                        onChangeCaptcha()
                        endTiming()
                        onResetForm(formRef)
                    }
                })
                .catch(() => {
                    state.submitRetrieveLoading = false
                })
        }
    })
}

const sendRegisterCaptcha = (formRef: FormInstance | undefined = undefined) => {
    if (state.codeSendCountdown > 0) return
    formRef!.validateField([state.form.registerType, 'username', 'password']).then((valid) => {
        if (valid) {
            state.sendCaptchaLoading = true
            const func = state.form.registerType == 'email' ? sendEms : sendSms
            func(state.form[state.form.registerType], 'user_register')
                .then((res) => {
                    if (res.code == 1) startTiming(60)
                })
                .finally(() => {
                    state.sendCaptchaLoading = false
                })
        }
    })
}

const sendRetrieveCaptcha = (formRef: FormInstance | undefined = undefined) => {
    if (state.codeSendCountdown > 0) return
    formRef!.validateField('account').then((valid) => {
        if (valid) {
            state.sendCaptchaLoading = true
            const func = state.retrievePasswordForm.type == 'email' ? sendEms : sendSms
            func(state.retrievePasswordForm.account, 'user_retrieve_pwd')
                .then((res) => {
                    if (res.code == 1) startTiming(60)
                })
                .finally(() => {
                    state.sendCaptchaLoading = false
                })
        }
    })
}

const startTiming = (seconds: number) => {
    state.codeSendCountdown = seconds
    timer = setInterval(() => {
        state.codeSendCountdown--
        if (state.codeSendCountdown <= 0) {
            endTiming()
        }
    }, 1000)
}
const endTiming = () => {
    state.codeSendCountdown = 0
    clearInterval(timer)
}

onMounted(async () => {
    if (await loginMounted()) return

    resize()
    useEventListener(window, 'resize', resize)

    // checkIn('get').then((res) => {
    //     state.accountVerificationType = res.data.accountVerificationType
    //     state.retrievePasswordForm.type = res.data.accountVerificationType.length > 0 ? res.data.accountVerificationType[0] : ''
    // })
})
onUnmounted(() => {
    state.codeSendCountdown = 0
    endTiming()
})
</script>

<style scoped lang="scss">
.login-box {
    width: 460px;
    margin: 40px auto;
    padding: 10px 60px 20px 60px;
    background-color: var(--ba-bg-color-overlay);
}

.login-title {
    text-align: center;
    font-size: var(--el-font-size-large);
    line-height: 100px;
    user-select: none;
}

:deep(.el-input--large) .el-input__wrapper {
    padding: 4px 15px;
}

.form-buttons {
    padding-top: 20px;

    .el-button {
        width: 100%;
        letter-spacing: 2px;
        font-weight: 300;
        margin-top: 20px;
        margin-left: 0;
    }
}

.register-verification-radio {
    margin-top: 10px;
}

.captcha-box {
    display: flex;
    align-items: center;
    justify-content: flex-end;

    .captcha-img {
        width: 90%;
        margin-left: auto;
    }

    .el-button {
        width: 90%;
        height: 100%;
    }
}

.form-footer {
    display: flex;
    align-items: center;

    .forgot-password {
        color: var(--ba-color-primary-light);
        margin-left: auto;
        user-select: none;
        cursor: pointer;
    }
}

.retrieve-password-form {
    display: flex;
    justify-content: center;
    margin-right: 50px;
}

@media screen and (max-width: 768px) {
    .login-box {
        width: 100%;
        margin: 0 auto;
    }
    .retrieve-password-form {
        margin-right: 0;
    }
}

// 暗黑样式
@at-root .dark {
    .form-buttons {
        .login-btn {
            --el-button-bg-color: var(--el-color-primary-light-5);
            --el-button-border-color: rgba(240, 252, 241, 0.1);
        }
    }

    .captcha-img {
        filter: brightness(61%);
    }
}
</style>
