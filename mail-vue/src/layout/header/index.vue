<template>
  <div class="header" :class="!hasPerm('email:send') ? 'not-send' : ''">
    <div class="header-btn">
      <hanburger @click="changeAside"></hanburger>
      <span class="breadcrumb-item">{{ $t(route.meta.title) }}</span>
    </div>
    <div v-perm="'email:send'" class="writer-box" @click="openSend">
      <div class="writer">
        <Icon icon="material-symbols:edit-outline-sharp" width="22" height="22"/>
      </div>
    </div>
    <div class="toolbar">
      <div v-if="uiStore.dark" class="sun-icon icon-item" @click="openDark($event)">
        <Icon icon="mingcute:sun-fill"/>
      </div>
      <div v-else class="dark-icon icon-item" @click="openDark($event)">
        <Icon icon="solar:moon-linear"/>
      </div>
      <div class="notice icon-item" @click="openNotice">
        <Icon icon="streamline-plump:announcement-megaphone"/>
      </div>
      <el-dropdown ref="userinfoRef" @visible-change="e => userInfoShow = e" :teleported="true" :popper-options="{ strategy: 'fixed' }" popper-class="detail-dropdown">
        <div class="avatar" @click="userInfoHide" >
          <div class="avatar-text">
            <div>{{ formatName(userStore.user.email) }}</div>
          </div>
          <Icon class="setting-icon" icon="mingcute:down-small-fill" width="24" height="24"/>
        </div>
        <template #dropdown>
          <div class="user-details">
            <div class="details-avatar">
              {{ formatName(userStore.user.email) }}
            </div>
            <div class="user-name">
              {{ userStore.user.name }}
            </div>
            <div class="detail-email" @click="copyEmail(userStore.user.email)">
              {{ userStore.user.email }}
            </div>
            <div class="detail-user-type">
              <el-tag>{{ userStore.user.role.name }}</el-tag>
            </div>
            <div class="action-info">
              <div class="quota-row">
                <span class="quota-label">{{ $t('sendCount') }}</span>
                <div class="quota-value">
                  <span v-if="sendCount" class="quota-count">{{ sendCount }}</span>
                  <el-tag round>{{ sendType }}</el-tag>
                </div>
              </div>
              <div class="quota-row">
                <span class="quota-label">{{ $t('accountCount') }}</span>
                <div class="quota-value">
                  <el-tag v-if="settingStore.settings.manyEmail || settingStore.settings.addEmail" round>
                    {{ $t('disabled') }}
                  </el-tag>
                  <span v-else-if="accountCount && hasPerm('account:add')" class="quota-count">
                    {{ $t('totalUserAccount', {msg: accountCount}) }}
                  </span>
                  <el-tag v-else-if="!accountCount && hasPerm('account:add')" round>{{ $t('unlimited') }}</el-tag>
                  <el-tag v-else-if="!hasPerm('account:add')" round>{{ $t('unauthorized') }}</el-tag>
                </div>
              </div>
            </div>
            <div class="logout">
              <el-button type="primary" :loading="logoutLoading" @click="clickLogout">{{ $t('logOut') }}</el-button>
            </div>
          </div>
        </template>
      </el-dropdown>
    </div>
  </div>
</template>

<script setup>
import router from "@/router";
import hanburger from '@/components/hamburger/index.vue'
import {logout} from "@/request/login.js";
import {Icon} from "@iconify/vue";
import {useUiStore} from "@/store/ui.js";
import {useUserStore} from "@/store/user.js";
import {useRoute} from "vue-router";
import {computed, ref} from "vue";
import {useSettingStore} from "@/store/setting.js";
import {hasPerm} from "@/perm/perm.js"
import {useI18n} from "vue-i18n";
import {setExtend} from "@/utils/day.js"

const {t} = useI18n();
const route = useRoute();
const settingStore = useSettingStore();
const userStore = useUserStore();
const uiStore = useUiStore();
const logoutLoading = ref(false)
const userInfoShow = ref(false)
const userinfoRef = ref({})

const accountCount = computed(() => {
  return userStore.user.role.accountCount
})

const sendType = computed(() => {

  if (settingStore.settings.send === 1) {
    return t('disabled')
  }

  if (!hasPerm('email:send')) {
    return t('unauthorized')
  }

  if (userStore.user.role.sendType === 'ban') {
    return t('sendBanned')
  }

  if (userStore.user.role.sendType === 'internal') {
    return t('sendInternal')
  }

  if (!userStore.user.role.sendCount) {
    return t('unlimited')
  }

  if (userStore.user.role.sendType === 'day') {
    return t('daily')
  }

  if (userStore.user.role.sendType === 'count') {
    return t('total')
  }
})

const sendCount = computed(() => {


  if (!hasPerm('email:send')) {
    return null
  }

  if (userStore.user.role.sendType === 'ban') {
    return null
  }

  if (userStore.user.role.sendType === 'internal') {
    return null
  }

  if (!userStore.user.role.sendCount) {
    return null
  }

  if (settingStore.settings.send === 1) {
    return null
  }

  return userStore.user.sendCount + '/' + userStore.user.role.sendCount
})

function userInfoHide(e) {
    if (userInfoShow.value) {
        userinfoRef.value.handleClose()
    } else {
        userinfoRef.value.handleOpen()
    }
}

async function copyEmail(email) {
  try {
    await navigator.clipboard.writeText(email);
    ElMessage({
      message: t('copySuccessMsg'),
      type: 'success',
      plain: true,
    })
  } catch (err) {
    console.error(`${t('copyFailMsg')}:`, err);
    ElMessage({
      message: t('copyFailMsg'),
      type: 'error',
      plain: true,
    })
  }
}

function changeLang(lang) {
  setExtend(lang === 'en' ? 'en' : 'zh-cn')
  settingStore.lang = lang
}

function openNotice() {
  uiStore.showNotice()
}

function openDark(e) {

  const nextIsDark = !uiStore.dark
  const root = document.documentElement

  if (!document.startViewTransition) {
    switchDark(nextIsDark, root);
    return
  }

  const x = e.clientX
  const y = e.clientY

  const maxX = Math.max(x, window.innerWidth - x)
  const maxY = Math.max(y, window.innerHeight - y)
  const endRadius = Math.hypot(maxX, maxY)

  // 标记切换目标，供 CSS 选择器使用
  root.setAttribute('data-theme-to', nextIsDark ? 'dark' : 'light')
  root.style.setProperty('--vt-x', `${x}px`)
  root.style.setProperty('--vt-y', `${y}px`)
  root.style.setProperty('--vt-end-radius', `${endRadius + 10}px`)

  const transition = document.startViewTransition(() => {
    switchDark(nextIsDark, root);
  })

  transition.finished.finally(() => {
    // 清理标记
    root.removeAttribute('data-theme-to')
  })
}

function switchDark(nextIsDark, root) {
  root.setAttribute('class', nextIsDark ? 'dark' : '')
  const metaTag = document.getElementById('theme-color-meta');
  const isMobile =  !window.matchMedia("(pointer: fine) and (hover: hover)").matches;
  metaTag.setAttribute('content', nextIsDark ? (isMobile ? '#141414' : '#000000') : (isMobile ? '#FFFFFF' : '#F1F1F1'));
  uiStore.dark = nextIsDark
}

function openSend() {
  uiStore.writerRef.open()
}

function changeAside() {
  uiStore.asideShow = !uiStore.asideShow
}

function clickLogout() {
  logoutLoading.value = true
  logout().then(() => {
    localStorage.removeItem("token")
    router.replace('/login')
  }).finally(() => {
    logoutLoading.value = false
  })
}

function formatName(email) {
  return email[0]?.toUpperCase() || ''
}

</script>
<style>
.detail-dropdown.el-popper,
.el-popper.detail-dropdown {
  --el-popper-border-radius: 24px;
  z-index: 5000 !important;
  padding: 0 !important;
  overflow: hidden;
  border: 1px solid color-mix(in srgb, var(--el-color-primary) 22%, var(--el-border-color)) !important;
  border-radius: 24px !important;
  background:
      radial-gradient(circle at 18% 0%, color-mix(in srgb, var(--el-color-primary) 18%, transparent), transparent 38%),
      radial-gradient(circle at 94% 18%, rgba(143, 113, 160, .16), transparent 34%),
      color-mix(in srgb, var(--el-bg-color) 94%, transparent) !important;
  box-shadow:
      0 24px 70px rgba(35, 49, 62, .22),
      0 4px 18px rgba(75, 94, 112, .12) !important;
  backdrop-filter: blur(24px) saturate(125%);
  -webkit-backdrop-filter: blur(24px) saturate(125%);
}

.detail-dropdown .el-popper__arrow::before {
  border-color: color-mix(in srgb, var(--el-color-primary) 18%, var(--el-border-color)) !important;
  background: color-mix(in srgb, var(--el-bg-color) 96%, transparent) !important;
}

.detail-dropdown .el-scrollbar,
.detail-dropdown .el-scrollbar__wrap,
.detail-dropdown .el-dropdown-menu,
.detail-dropdown .el-dropdown__list {
  margin: 0 !important;
  padding: 0 !important;
  border: 0 !important;
  background: transparent !important;
}
</style>
<style lang="scss" scoped>

.user-details {
  position: relative;
  width: min(340px, calc(100vw - 28px));
  max-width: 100%;
  padding: 26px 22px 20px;
  box-sizing: border-box;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0;
  font-size: 14px;
  color: var(--el-text-color-primary);
}

.user-details::before {
  content: '';
  position: absolute;
  width: 170px;
  height: 170px;
  left: -72px;
  top: -86px;
  border-radius: 50%;
  pointer-events: none;
  background: color-mix(in srgb, var(--el-color-primary) 16%, transparent);
  filter: blur(2px);
}

.user-details::after {
  content: '';
  position: absolute;
  width: 150px;
  height: 150px;
  right: -78px;
  top: 48px;
  border-radius: 50%;
  pointer-events: none;
  background: rgba(126, 156, 137, .12);
}

.details-avatar,
.user-name,
.detail-email,
.detail-user-type,
.action-info,
.logout {
  position: relative;
  z-index: 1;
}

.user-details .details-avatar {
  width: 64px;
  height: 64px;
  margin: 0 0 14px;
  display: grid;
  place-items: center;
  border-radius: 22px;
  border: 1px solid color-mix(in srgb, var(--el-color-primary) 30%, var(--el-border-color));
  color: var(--el-color-primary);
  background:
      linear-gradient(145deg,
        color-mix(in srgb, var(--el-color-primary-light-8) 88%, var(--el-bg-color)),
        color-mix(in srgb, #9a8cad 22%, var(--el-bg-color)));
  box-shadow: 0 12px 28px rgba(67, 87, 105, .17);
  font-size: 26px;
  font-weight: 800;
  letter-spacing: .5px;
}

.user-details .user-name {
  width: 100%;
  padding: 0 12px;
  box-sizing: border-box;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  text-align: center;
  font-size: 20px;
  line-height: 1.35;
  font-weight: 760;
  letter-spacing: .2px;
}

.user-details .detail-email {
  max-width: 100%;
  margin-top: 6px;
  padding: 7px 13px;
  box-sizing: border-box;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  text-align: center;
  border-radius: 999px;
  color: var(--el-text-color-secondary);
  background: color-mix(in srgb, var(--el-fill-color-light) 72%, transparent);
  cursor: pointer;
  transition: color .18s ease, background .18s ease, transform .18s ease;
}

.user-details .detail-email:hover {
  color: var(--el-color-primary);
  background: var(--el-color-primary-light-9);
  transform: translateY(-1px);
}

.user-details .detail-user-type {
  margin-top: 10px;
}

.user-details .detail-user-type :deep(.el-tag) {
  min-height: 28px;
  padding-inline: 13px;
  border-radius: 999px;
  border-color: color-mix(in srgb, var(--el-color-primary) 25%, var(--el-border-color));
  background: color-mix(in srgb, var(--el-color-primary-light-9) 76%, transparent);
}

.user-details .action-info {
  width: 100%;
  margin-top: 20px;
  padding: 10px;
  box-sizing: border-box;
  display: flex;
  flex-direction: column;
  gap: 8px;
  border: 1px solid color-mix(in srgb, var(--el-color-primary) 14%, var(--el-border-color));
  border-radius: 18px;
  background: color-mix(in srgb, var(--el-fill-color-light) 58%, transparent);
}

.user-details .quota-row {
  min-height: 48px;
  padding: 9px 11px;
  box-sizing: border-box;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  border-radius: 13px;
  background: color-mix(in srgb, var(--el-bg-color) 80%, transparent);
}

.user-details .quota-label {
  flex: 0 0 auto;
  color: var(--el-text-color-regular);
  font-weight: 650;
  white-space: nowrap;
}

.user-details .quota-value {
  min-width: 0;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  flex-wrap: wrap;
  gap: 7px;
  text-align: right;
}

.user-details .quota-count {
  max-width: 150px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: var(--el-text-color-secondary);
  font-size: 13px;
}

.user-details .quota-value :deep(.el-tag) {
  min-height: 28px;
  padding-inline: 12px;
  border-radius: 999px;
  color: color-mix(in srgb, var(--el-color-primary) 80%, var(--el-text-color-primary));
  border-color: color-mix(in srgb, var(--el-color-primary) 24%, var(--el-border-color));
  background: color-mix(in srgb, var(--el-color-primary-light-9) 75%, transparent);
}

.user-details .logout {
  width: 100%;
  margin-top: 16px;
}

.user-details .logout :deep(.el-button) {
  width: 100%;
  height: 46px;
  margin: 0;
  border: 0;
  border-radius: 16px;
  color: #fff;
  font-size: 15px;
  font-weight: 700;
  letter-spacing: .3px;
  background: linear-gradient(110deg, #7195ad, #9588aa 52%, #7aa08e);
  box-shadow: 0 12px 28px rgba(77, 95, 121, .22);
  transition: transform .18s ease, box-shadow .18s ease, filter .18s ease;
}

.user-details .logout :deep(.el-button:hover) {
  transform: translateY(-1px);
  box-shadow: 0 16px 34px rgba(77, 95, 121, .28);
  filter: saturate(1.06) brightness(1.03);
}

@media (max-width: 420px) {
  .user-details {
    width: min(320px, calc(100vw - 20px));
    padding: 22px 16px 16px;
  }

  .user-details .action-info {
    padding: 8px;
  }

  .user-details .quota-row {
    padding: 8px 9px;
    gap: 9px;
  }
}


.header {
  text-align: right;
  font-size: 12px;
  display: grid;
  height: 100%;
  gap: 10px;
  grid-template-columns: auto auto 1fr;
}

.header.not-send {
  grid-template-columns: auto 1fr;
}

.writer-box {
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-left: 5px;

  .writer {
    width: 34px;
    height: 34px;
    border-radius: 50%;
    color: #ffffff;
    background: linear-gradient(135deg, #1890ff, #3a80dd);
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    justify-content: center;

    .writer-text {
      margin-left: 15px;
      font-size: 14px;
      font-weight: bold;;
    }
  }
}

.header-btn {
  display: inline-flex;
  align-items: center;
  height: 100%;
  min-width: 0;
}

.breadcrumb-item {
  font-weight: bold;
  font-size: 14px;
  color: var(--el-text-color-primary);
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}

.toolbar {
  display: flex;
  justify-content: end;
  gap: 15px;
  @media (max-width: 767px) {
    gap: 10px;
  }

  .icon-item {
    align-self: center;
    width: 30px;
    height: 30px;
    border-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
  }

  .icon-item:hover {
    background: var(--base-fill);
  }

  .notice {
    font-size: 22px;
    margin-right: 4px;
  }

  .dark-icon {
    font-size: 20px;
  }

  .sun-icon {
    font-size: 24px;
  }

  .avatar {
    display: flex;
    align-items: center;
    cursor: pointer;

    .avatar-text {
      background: var(--el-bg-color);
      color: var(--el-text-color-primary);
      height: 30px;
      width: 30px;
      display: flex;
      justify-content: center;
      align-items: center;
      border-radius: 8px;
      border: 1px solid var(--dark-border);
    }

    .setting-icon {
      position: relative;
      top: 0;
      margin-right: 10px;
      bottom: 10px;
    }
  }

}

.el-tooltip__trigger:first-child:focus-visible {
  outline: unset;
}


.header {
  padding: 0 10px 0 8px;
  border: 1px solid color-mix(in srgb, var(--el-border-color) 76%, transparent);
  border-radius: 18px;
  background: color-mix(in srgb, var(--el-bg-color) 82%, transparent);
  box-shadow: 0 12px 34px rgba(57, 79, 96, .10);
  backdrop-filter: blur(18px) saturate(115%);
}
.breadcrumb-item {
  font-size: 15px;
  letter-spacing: .15px;
}
.writer-box .writer {
  width: 38px;
  height: 38px;
  border-radius: 13px;
  background: linear-gradient(135deg, #6d93ad, #9488ae 56%, #76a08d);
  box-shadow: 0 9px 20px rgba(83, 104, 130, .24);
}
.writer-box:hover .writer { transform: translateY(-1px) scale(1.03); }
.toolbar .icon-item {
  width: 34px;
  height: 34px;
  border-radius: 12px;
  transition: background .18s ease, transform .18s ease;
}
.toolbar .icon-item:hover {
  transform: translateY(-1px);
  background: color-mix(in srgb, var(--el-color-primary-light-9) 78%, transparent);
}
.toolbar .avatar .avatar-text {
  height: 34px;
  width: 34px;
  border-radius: 12px;
  border-color: color-mix(in srgb, var(--el-color-primary) 28%, var(--el-border-color));
  background: linear-gradient(145deg, var(--el-bg-color), var(--el-fill-color-light));
  font-weight: 750;
}
</style>
