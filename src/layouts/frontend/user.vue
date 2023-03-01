<template>
    <component :is="memberCenter.state.layoutMode"></component>
</template>

<script setup lang="ts">
import {onMounted} from 'vue'
import {useUserInfo} from '/@/stores/userInfo'
import {useMemberCenter} from '/@/stores/memberCenter'
import {index} from '/@/api/frontend/user/index'
import {handleMemberCenterRoute, getFirstRoute, routePush} from '/@/utils/router'
import {memberCenterBaseRoute} from '/@/router/static'
import {ElNotification} from 'element-plus'
import {useI18n} from 'vue-i18n'
import {useRoute, useRouter, onBeforeRouteUpdate} from 'vue-router'
import userMounted from '/@/components/mixins/userMounted'
import {isEmpty} from 'lodash-es'

const {t} = useI18n()
const route = useRoute()
const router = useRouter()
const userInfo = useUserInfo()
const memberCenter = useMemberCenter()

onBeforeRouteUpdate((to) => {
    memberCenter.setActiveRoute(to)
})

onMounted(async () => {
    const ret = await userMounted()
    if (ret.type == 'break') return
    if (ret.type == 'reload') return (window.location.href = ret.url)

    if (!userInfo.token) return router.push({name: 'userLogin'})


    //res.data.userInfo.refreshToken = userInfo.refreshToken
    //userInfo.dataFill(res.data.userInfo)

    const menus = [
        {
            "id": 2,
            "pid": 0,
            "type": "menu_dir",
            "title": "我的账户",
            "name": "account",
            "path": "account",
            "icon": "fa fa-user-circle",
            "menu_type": "tab",
            "url": "",
            "component": "",
            "extend": "none",
            "children": [
                {
                    "id": 3,
                    "pid": 2,
                    "type": "menu",
                    "title": "账户概览",
                    "name": "account\/overview",
                    "path": "account\/overview",
                    "icon": "fa fa-home",
                    "menu_type": "tab",
                    "url": "",
                    "component": "\/src\/views\/frontend\/user\/account\/overview.vue",
                    "extend": "none"
                },
                {
                    "id": 4,
                    "pid": 2,
                    "type": "menu",
                    "title": "个人资料",
                    "name": "account\/profile",
                    "path": "account\/profile",
                    "icon": "fa fa-user-circle-o",
                    "menu_type": "tab",
                    "url": "",
                    "component": "\/src\/views\/frontend\/user\/account\/profile.vue",
                    "extend": "none"
                },
                {
                    "id": 5,
                    "pid": 2,
                    "type": "menu",
                    "title": "修改密码",
                    "name": "account\/changePassword",
                    "path": "account\/changePassword",
                    "icon": "fa fa-shield",
                    "menu_type": "tab",
                    "url": "",
                    "component": "\/src\/views\/frontend\/user\/account\/changePassword.vue",
                    "extend": "none"
                },
                {
                    "id": 6,
                    "pid": 2,
                    "type": "menu",
                    "title": "积分记录",
                    "name": "account\/integral",
                    "path": "account\/integral",
                    "icon": "fa fa-tag",
                    "menu_type": "tab",
                    "url": "",
                    "component": "\/src\/views\/frontend\/user\/account\/integral.vue",
                    "extend": "none"
                },
                {
                    "id": 7,
                    "pid": 2,
                    "type": "menu",
                    "title": "余额记录",
                    "name": "account\/balance",
                    "path": "account\/balance",
                    "icon": "fa fa-money",
                    "menu_type": "tab",
                    "url": "",
                    "component": "\/src\/views\/frontend\/user\/account\/balance.vue",
                    "extend": "none"
                }
            ]
        }
    ]
    handleMemberCenterRoute(menus)

    if (ret.type == 'jump') return router.push(ret.url)

    // 预跳转到上次路径
    if (route.params.to) {
        const lastRoute = JSON.parse(route.params.to as string)
        if (lastRoute.path != memberCenterBaseRoute.path) {
            let query = !isEmpty(lastRoute.query) ? lastRoute.query : {}
            routePush({path: lastRoute.path, query: query})
            return
        }
    }

    // 跳转到第一个菜单
    let firstRoute = getFirstRoute(memberCenter.state.viewRoutes)
    if (firstRoute) {
        router.push({path: firstRoute.path})
    } else {
        ElNotification({
            type: 'error',
            message: t('No route found to jump~'),
        })
    }

    if (document.body.clientWidth < 1024) {
        memberCenter.setShrink(true)
    } else {
        memberCenter.setShrink(false)
    }
})
</script>

<!-- 只有在 components 选项中的组件可以被动态组件使用-->
<script lang="ts">
import Default from '/@/layouts/frontend/container/default.vue'
import Disable from '/@/layouts/frontend/container/disable.vue'

export default {
    components: {Default, Disable},
}
</script>

<style scoped lang="scss"></style>
