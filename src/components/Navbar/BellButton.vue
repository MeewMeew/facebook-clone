<script setup lang="ts">
import { Popover, PopoverButton, PopoverPanel } from '@headlessui/vue'
import { formatTimeAgo } from '@vueuse/core'
import { useMemoize } from '@vueuse/core'
//@ts-ignore
import { useSound } from '@vueuse/sound'
import { uniqBy } from 'lodash'
import { storeToRefs } from 'pinia'
import Avatar from 'primevue/avatar'
import Badge from 'primevue/badge'
import Button from 'primevue/button'
import Skeleton from 'primevue/skeleton'
import InfiniteLoading from 'v3-infinite-loading'
import { computed, onMounted, ref } from 'vue'

import Bell from '@/components/Icons/Navbar/Bell.vue'
import { Attachment } from '@/database'
import { Notification } from '@/database'
import { User } from '@/database/user'
import { Logger } from '@/helpers/logger'
import { mewSocket } from '@/helpers/socket'
import { defaultFormatTimeMessages } from '@/shared/constants'
import { REACTIONS } from '@/shared/constants'
import { useGeneral } from '@/stores/general'
import { useUser } from '@/stores/user'
import type { InfiniteState, INotification, IUser, ReactionType } from '@/types'
import { NotificationType, SEvent } from '@/types'

type EXINotification = INotification & {
  user: IUser
}

const { play } = useSound('/sound/noti.mp3', { volume: 0.1 })
const { cuser } = storeToRefs(useUser())
const { notiCount } = storeToRefs(useGeneral())

const limit = 5
const lastDocument = ref<number>(0)
const notifications = ref<EXINotification[]>([])
const hasNoti = computed(() => notifications.value.length > 0)

const getUser = useMemoize((uid: number) => {
  return User.get({ id: uid })
})

const updateReadState = async (nid: string) => {
  await Notification.read(nid)

  const index = notifications.value.findIndex((n) => n.nid === nid)
  if (index !== -1) {
    notifications.value[index].read = true
    notiCount.value--
  }
}

const markAsReadAll = async () => {
  await Notification.readAll(cuser.value!.id)
  notifications.value.forEach((n) => (n.read = true))
  notiCount.value = 0
}

const loadNotifications = async function ($state: InfiniteState) {
  try {
    let data = (await Notification.getAll({
      aid: cuser.value?.id,
      limit: limit + lastDocument.value,
      sort: 'created_at',
      order: 'desc'
    })) as EXINotification[]

    for (let item of data) {
      const user = await getUser(item.data.uid)
      if (!user) {
        Logger.error('Bell user not found', item)
        await Notification.remove(item.nid)
        data = data.filter((n) => n.nid !== item.nid)
        continue
      }
      if (Attachment.isID(user.photoURL)) {
        const attachment = await Attachment.get(user.photoURL)
        user.photoURL = await Attachment.cacheImage(attachment.attachments.medium)
      }
      item.user = user
    }

    notifications.value = uniqBy([...data], 'nid')

    if ((data.length <= lastDocument.value && lastDocument.value !== 0) || data.length === 0) {
      $state.complete()
    } else {
      notiCount.value = notifications.value.filter((n) => !n.read).length
      $state.loaded()
    }
    lastDocument.value = notifications.value.length
  } catch (error) {
    Logger.error('Bell', error)
    $state.error()
  }
}

const getReaction = (name: ReactionType) => {
  return REACTIONS.find((r) => r.name === name)
}

onMounted(() => {
  Logger.info('Bell mounted', notifications.value)

  mewSocket.on(SEvent.NOTIFICATION_CREATE, async (data: INotification) => {
    const user = await User.get({ id: data.data.uid })
    if (!user) return
    if (Attachment.isID(user.photoURL)) {
      const attachment = await Attachment.get(user.photoURL)
      user.photoURL = await Attachment.cacheImage(attachment.attachments.medium)
    }
    play()

    notifications.value.unshift({ ...data, user })
  })

  mewSocket.on(SEvent.NOTIFICATION_REMOVE, async (data: INotification) => {
    const index = notifications.value.findIndex((n) => n.nid === data.nid)
    if (index !== -1) {
      notifications.value.splice(index, 1)
    }
  })

  mewSocket.on(SEvent.NOTIFICATION_UPDATE, (data: INotification) => {
    const index = notifications.value.findIndex((n) => n.nid === data.nid)
    if (index !== -1) {
      notifications.value[index].data = data.data
    }
  })
})
</script>

<template>
  <Popover class="relative" v-slot="{ open }">
    <PopoverButton
      class="rounded-full bg-[#E3E6EA] p-2 hover:bg-gray-300 mx-1 cursor-pointer relative focus:ring-0 focus:ring-offset-0 ring-0"
    >
      <Bell size="22" :fill="open" />
      <Badge
        v-show="notiCount > 0"
        :value="notiCount"
        class="absolute -top-1 -right-1"
        severity="danger"
        :pt="{
          root: 'rounded-full p-0 text-center inline-block bg-blue-500 text-white font-bold bg-red-500 text-[10px] min-w-[1rem] h-[1rem] leading-[1rem]'
        }"
      />
    </PopoverButton>
    <Transition
      enter-active-class="transition duration-200 ease-out"
      enter-from-class="translate-y-1 opacity-0"
      enter-to-class="translate-y-0 opacity-100"
      leave-active-class="transition duration-150 ease-in"
      leave-from-class="translate-y-0 opacity-100"
      leave-to-class="translate-y-1 opacity-0"
    >
      <PopoverPanel
        class="absolute -left-28 z-10 mt-3 -translate-x-1/2 transform px-4 sm:px-0 lg:max-w-xl border bg-white rounded-lg max-h-[80vh] w-fit py-3 overflow-y-scroll nice-scrollbar"
      >
        <div class="px-3 flex flex-row justify-between items-center pb-2">
          <div class="text-xl font-bold">
            <span class="">Thông báo</span>
          </div>
          <Button
            label="Read all"
            text
            :pt="{ label: 'text-sm' }"
            size="small"
            @click="markAsReadAll"
          />
        </div>
        <div
          class="flex flex-col w-[350px] justify-center items-center animate__animated animate__fadeIn transition-all px-2"
        >
          <router-link
            :to="
              item.type.includes('friend')
                ? { name: 'user', params: { id: item.data.uid } }
                : { name: 'post', params: { id: item.data.pid } }
            "
            v-show="hasNoti"
            class="relative items-center hover:bg-gray-100 cursor-pointer rounded-lg duration-300 w-full"
            v-for="(item, index) in notifications"
            :key="index"
            @click="updateReadState(item.nid)"
          >
            <div class="flex flex-row justify-start items-center h-16 w-full p-2 space-x-3">
              <div class="relative">
                <Avatar
                  :image="item.user.photoURL"
                  :pt="{ root: 'w-12 h-12', image: 'rounded-full w-full h-full' }"
                />
                <div class="absolute -right-1 -bottom-1">
                  <img
                    v-if="item.type === NotificationType.POST_COMMENT"
                    src="/icons/noti/comment.png"
                    alt=""
                    class="w-6 h-6"
                  />
                  <img
                    v-if="item.type.includes('friend')"
                    src="/icons/noti/friend.png"
                    alt=""
                    class="w-6 h-6"
                  />
                  <img
                    v-if="item.type === NotificationType.POST_REACTION"
                    :src="getReaction(item.data.type as ReactionType)?.icon"
                    class="w-6 h-6"
                  />
                </div>
              </div>
              <div class="">
                <div class="flex flex-col">
                  <div class="text-sm">
                    <router-link
                      :to="{ name: 'user', params: { id: item.user.id } }"
                      class="hover:underline"
                    >
                      <span class="font-medium">{{ item.user.displayName }}</span>
                    </router-link>
                    <span v-if="item.type === NotificationType.POST_COMMENT">
                      đã bình luận về bài viết của bạn.</span
                    >
                    <span v-if="item.type === NotificationType.POST_REACTION">
                      đã bày tỏ cảm xúc về bài viết của bạn.</span
                    >
                    <span v-if="item.type === NotificationType.FRIEND_RECEIVE">
                      đã gửi lời mời kết bạn.</span
                    >
                    <span v-if="item.type === NotificationType.FRIEND_ACCEPT">
                      đã chấp nhận lời mời kết bạn.</span
                    >
                    <span v-if="item.type === NotificationType.FRIEND_REJECT">
                      đã từ chối lời mời kết bạn.</span
                    >
                    <span v-if="item.type === NotificationType.FRIEND_REMOVE">
                      đã huỷ kết bạn với bạn.</span
                    >
                  </div>
                  <div class="text-mb-blue text-xs">
                    <span>{{
                      formatTimeAgo(new Date(item.created_at), {
                        messages: defaultFormatTimeMessages
                      })
                    }}</span>
                  </div>
                </div>
              </div>
            </div>
            <span
              v-if="!item.read"
              class="absolute h-3 w-3 rounded-full bg-mb-blue top-1/2 right-0 -translate-x-1/2 -translate-y-1/2"
            />
          </router-link>
          <div v-show="!hasNoti">
            <span class="text-center text-mb-blue">Bạn không có thông báo nào</span>
          </div>
          <InfiniteLoading @infinite="loadNotifications">
            <template #spinner>
              <div class="flex flex-row">
                <Skeleton shape="circle" size="1rem" class="mx-1"></Skeleton>
                <Skeleton shape="circle" size="1rem" class="mx-1"></Skeleton>
                <Skeleton shape="circle" size="1rem" class="mx-1"></Skeleton>
              </div>
            </template>
            <template #complete>
              <div></div>
            </template>
            <template #error>
              <div></div>
            </template>
          </InfiniteLoading>
        </div>
      </PopoverPanel>
    </Transition>
  </Popover>
</template>
