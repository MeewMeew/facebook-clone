<script setup lang="ts">
import { Dialog, DialogPanel, DialogTitle, TransitionChild, TransitionRoot } from '@headlessui/vue'
import { storeToRefs } from 'pinia'
import Button from 'primevue/button'
import Image from 'primevue/image'
import { onBeforeMount, type PropType, reactive, ref } from 'vue'
import Camera from 'vue-material-design-icons/Camera.vue'
import { useRouter } from 'vue-router'

import CoverPhoto from '@/components/User/CoverPhoto.vue'
import { Attachment, Friend } from '@/database'
import { Logger } from '@/helpers/logger'
import { mewSocket } from '@/helpers/socket'
import { useGeneral } from '@/stores/general'
import { useUser } from '@/stores/user'
import { type IFriend, type IUser, NotificationType, SEvent } from '@/types'

const router = useRouter()
const props = defineProps({
  user: {
    type: Object as PropType<IUser>,
    required: true
  },
  tab: {
    type: String,
    default: 'user',
    required: false
  }
})

const user = ref<IUser>(props.user)
const fuser = ref<IFriend>()
const fcuser = ref<IFriend>()
const loading = reactive({
  edit: false,
  add: false,
  accept: false,
  cancel: false,
  reject: false,
  remove: false
})

const open = reactive({
  unfriend: false
})

const openDialogUF = () => (open.unfriend = true)
const closeDialogUF = () => (open.unfriend = false)

async function addFriend() {
  try {
    loading.add = true
    const event = await Friend.add(cuser.value!.id, user.value!.id)
    if (event) {
      mewSocket.emit(SEvent.FRIEND_REQUEST, event)
      fuser.value!.requested = [...fuser.value!.requested, cuser.value!.id.toString()]
      is.requested = true
    }
  } catch (error) {
    Logger.error(error)
  } finally {
    loading.add = false
  }
}

async function unfriend() {
  try {
    loading.remove = true
    const event = await Friend.remove(cuser.value!.id, user.value!.id)
    if (event) {
      mewSocket.emit(SEvent.FRIEND_REMOVE, event)
      fuser.value!.friends = fuser.value!.friends.filter((i) => i !== cuser.value!.id.toString())
      is.friend = false
    }
  } catch (error) {
    Logger.error(error)
  } finally {
    closeDialogUF()
    loading.remove = false
  }
}
async function cancelRequest() {
  try {
    loading.cancel = true
    const event = await Friend.cancel(cuser.value!.id, user.value!.id)
    if (event) {
      mewSocket.emit(SEvent.FRIEND_CANCEL, event)
      fuser.value!.received = fuser.value!.received.filter((i) => i !== cuser.value!.id.toString())
      is.requested = false
    }
  } catch (error) {
    Logger.error(error)
  } finally {
    loading.cancel = false
  }
}
async function acceptRequest() {
  try {
    loading.accept = true
    const event = await Friend.accept(cuser.value!.id, user.value!.id)
    if (event) {
      mewSocket.emit(SEvent.FRIEND_ACCEPT, event)
      fuser.value!.friends = [...fuser.value!.friends, cuser.value!.id.toString()]
      is.friend = true
      is.received = false
    }
  } catch (error) {
    Logger.error(error)
  } finally {
    loading.accept = false
  }
}
async function rejectRequest() {
  try {
    loading.reject = true
    const event = await Friend.reject(cuser.value!.id, user.value!.id)
    if (event) {
      mewSocket.emit(SEvent.FRIEND_REJECT, event)
      fuser.value!.received = fuser.value!.received.filter((i) => i !== cuser.value!.id.toString())
      is.received = false
    }
  } catch (error) {
    Logger.error(error)
  } finally {
    loading.reject = false
  }
}

const is = reactive({
  friend: false,
  requested: false,
  received: false
})

const { cuser } = storeToRefs(useUser())
const { isCropperModal, notiCount } = storeToRefs(useGeneral())
const id = +router.currentRoute.value.params.id

onBeforeMount(async () => {
  const f1 = await Friend.getByUID(id)
  const f2 = await Friend.getByUID(cuser.value!.id)
  if (f1) fuser.value = f1
  if (f2) fcuser.value = f2

  is.friend = fcuser.value?.friends.includes(fuser.value!.uid.toString())!
  is.requested = fcuser.value?.requested.includes(fuser.value!.uid.toString())!
  is.received = fcuser.value?.received.includes(fuser.value!.uid.toString())!

  if (Attachment.isID(user.value?.photoURL!)) {
    const attachment = await Attachment.get(user.value?.photoURL!)
    user.value!.photoURL = await Attachment.cacheImage(attachment.attachments.large)
  }

  if (Attachment.isID(user.value?.coverURL!)) {
    const attachment = await Attachment.get(user.value?.coverURL!)
    user.value!.coverURL = await Attachment.cacheImage(attachment.attachments.large)
  }

  mewSocket.on(SEvent.NOTIFICATION_CREATE, (data) => {
    if (data.aid === data.data.uid) return
    if (!fuser.value) return
    switch (data.type) {
    case NotificationType.FRIEND_RECEIVE:
      is.received = true
      fuser.value.received = [...fuser.value.received, cuser.value!.id.toString()]
      break

    case NotificationType.FRIEND_ACCEPT:
      is.friend = true
      is.received = false
      is.requested = false
      fuser.value.friends = [...fuser.value.friends, cuser.value!.id.toString()]
      break

    case NotificationType.FRIEND_REMOVE:
      is.friend = false
      fuser.value.friends = fuser.value.friends.filter((i) => i !== cuser.value!.id.toString())
      break

    case NotificationType.FRIEND_REJECT:
      is.requested = false
      fuser.value.requested = fuser.value.requested.filter(
        (i) => i !== cuser.value!.id.toString()
      )
      break
    default:
      break
    }
  })
  mewSocket.on(SEvent.NOTIFICATION_REMOVE, (data) => {
    if (data.aid === data.data.uid) return
    if (!fuser.value) return

    if (data.type === NotificationType.FRIEND_RECEIVE) {
      is.received = false
      fuser.value.received = fuser.value.received.filter((i) => i !== cuser.value!.id.toString())
      notiCount.value--
    }
  })
})
</script>
<template>
  <div class="w-full bg-white" v-if="cuser && user">
    <div class="max-w-xl mx-auto pb-1">
      <CoverPhoto :cover="user.coverURL" :uid="user.id" />

      <div class="flex md:flex-row flex-col items-center justify-between px-4 pb-4 md:pb-0">
        <div class="flex md:flex-row flex-col gap-4 md:-mt-10 -mt-16 items-center">
          <div class="relative">
            <Image
              :src="user.photoURL || Attachment.defaultAvatar"
              preview
              :pt="{
                image: {
                  class: 'rounded-full cursor-pointer border-4 border-white w-[180px] h-[180px]'
                },
                button: {
                  class: [
                    'absolute inset-0 flex items-center justify-center opacity-0 transition-opacity duration-300 ',
                    'bg-transparent text-gray-100',
                    'hover:opacity-20 hover:cursor-pointer hover:bg-black hover:bg-opacity-50',
                    'rounded-full'
                  ]
                },
                icon: {
                  class: 'hidden opacity-0'
                }
              }"
            />
            <button
              v-if="cuser.id === user.id"
              @click="isCropperModal = true"
              class="absolute right-1 bottom-5 bg-gray-200 hover:bg-gray-300 p-1.5 rounded-full cursor-pointer"
            >
              <Camera :size="25" />
            </button>
          </div>
          <div class="md:mt-4 text-center md:text-left -mt-3">
            <div class="text-[28px] font-bold pt-1">
              {{ user.displayName }}
            </div>
            <div
              v-if="fuser"
              class="text-md font-semibold text-gray-600 mb-1.5 text-center md:text-left"
            >
              {{ fuser.friends.length }} bạn bè
            </div>
          </div>
        </div>

        <Button
          v-if="cuser.id === user.id"
          aria-label="Chỉnh sửa trang cá nhân"
          severity="secondary"
          class="space-x-2 bg-mb-gray hover:bg-mb-gray-h border-mb-gray"
          size="small"
        >
          <i class="pi pi-pencil text-gray-700"></i>
          <span class="font-medium text-[16px] text-gray-700">Chỉnh sửa trang cá nhân</span>
        </Button>
        <div v-else>
          <Button
            v-if="is.friend"
            aria-label="Bạn bè"
            class="space-x-2"
            size="small"
            :loading="loading.remove"
            @click="openDialogUF"
          >
            <i v-if="loading.remove" class="pi pi-spin pi-spinner"></i>
            <img v-else src="/icons/friend/is-friend.png" class="invert w-4 h-4" />
            <span class="font-medium text-[16px]">Bạn bè</span>
          </Button>
          <TransitionRoot appear :show="open.unfriend" as="template">
            <Dialog as="div" @close="closeDialogUF" class="relative z-10">
              <TransitionChild
                as="template"
                enter="duration-300 ease-out"
                enter-from="opacity-0"
                enter-to="opacity-100"
                leave="duration-200 ease-in"
                leave-from="opacity-100"
                leave-to="opacity-0"
              >
                <div class="fixed inset-0 bg-black bg-opacity-25" />
              </TransitionChild>

              <div class="fixed inset-0 overflow-y-auto">
                <div class="flex min-h-full items-center justify-center p-4 text-center">
                  <TransitionChild
                    as="template"
                    enter="duration-300 ease-out"
                    enter-from="opacity-0 scale-95"
                    enter-to="opacity-100 scale-100"
                    leave="duration-200 ease-in"
                    leave-from="opacity-100 scale-100"
                    leave-to="opacity-0 scale-95"
                  >
                    <DialogPanel
                      class="w-full max-w-[550px] transform overflow-hidden rounded-2xl bg-white p-6 text-left align-middle shadow-xl transition-all"
                    >
                      <DialogTitle
                        as="h3"
                        class="text-lg font-medium leading-6 text-gray-900 text-center border-b pb-4"
                      >
                        Huỷ kết bạn với {{ user.displayName }}
                      </DialogTitle>
                      <div class="mt-2">
                        <p class="text-md text-gray-500">
                          Bạn có chắc muốn huỷ kết bạn với {{ user.displayName }}?
                        </p>
                      </div>

                      <div class="mt-4 flex justify-end items-center space-x-2">
                        <Button label="Huỷ" text @click="closeDialogUF" />
                        <Button label="Xác nhận" @click="unfriend" />
                      </div>
                    </DialogPanel>
                  </TransitionChild>
                </div>
              </div>
            </Dialog>
          </TransitionRoot>
          <Button
            v-if="is.requested"
            aria-label="Huỷ lời mời"
            class="space-x-2"
            size="small"
            :loading="loading.cancel"
            @click="cancelRequest"
          >
            <i v-if="loading.cancel" class="pi pi-spin pi-spinner"></i>
            <img v-else src="/icons/friend/cancel-add-friend.png" class="invert w-4 h-4" />
            <span class="font-medium text-[16px]">Huỷ lời mời</span>
          </Button>
          <div v-if="is.received" class="flex flex-row items-center justify-center space-x-3">
            <Button
              aria-label="Chấp nhận"
              class="space-x-2"
              size="small"
              :loading="loading.accept"
              @click="acceptRequest"
            >
              <i v-if="loading.accept" class="pi pi-spin pi-spinner"></i>
              <img v-else src="/icons/friend/add-friend.png" class="invert w-4 h-4" />
              <span class="font-medium text-[16px]">Chấp nhận</span>
            </Button>
            <Button
              aria-label="Từ chối"
              class="space-x-2 bg-mb-gray hover:bg-mb-gray-h border-mb-gray"
              size="small"
              :loading="loading.reject"
              @click="rejectRequest"
            >
              <i v-if="loading.reject" class="pi pi-spin pi-spinner text-gray-700"></i>
              <img v-else src="/icons/friend/cancel-add-friend.png" class="w-4 h-4" />
              <span class="font-medium text-[16px] text-gray-700">Từ chối</span>
            </Button>
          </div>
          <Button
            v-if="!is.friend && !is.requested && !is.received"
            aria-label="Thêm bạn bè"
            class="space-x-2"
            size="small"
            :loading="loading.add"
            @click="addFriend"
          >
            <i v-if="loading.add" class="pi pi-spin pi-spinner"></i>
            <img v-else src="/icons/friend/add-friend.png" class="invert w-4 h-4" />
            <span class="font-medium text-[16px]">Thêm bạn bè</span>
          </Button>
        </div>
      </div>

      <div class="flex items-centerw-full border-t h-[50px] mb-2 pt-2 transition-all">
        <div class="w-[85px] duration-150">
          <router-link
            :to="{ name: 'user', params: { id: user.id } }"
            class="flex items-center text-[15px] justify-center h-[48px] hover:bg-[#F2F2F2] font-medium rounded-lg cursor-pointer"
            :class="[tab === 'user' ? 'text-blue-500' : '']"
          >
            Bài viết
          </router-link>
          <div v-if="tab === 'user'" class="border-t-4 border-t-blue-400 rounded-md w-full"></div>
        </div>
        <button
          class="flex items-center text-[15px] justify-center h-[48px] p-1 hover:bg-[#F2F2F2] font-medium rounded-lg mx-1 cursor-pointer"
        >
          Giới thiệu
        </button>
        <div class="w-[85px] duration-150">
          <router-link
            :to="{ name: 'fuser' }"
            class="flex items-center text-[15px] justify-center h-[48px] p-1 hover:bg-[#F2F2F2] font-medium rounded-lg mx-1 cursor-pointer"
            :class="[tab === 'fuser' ? 'text-blue-500' : '']"
          >
            Bạn bè
          </router-link>
          <div v-if="tab === 'fuser'" class="border-t-4 border-t-blue-400 rounded-md w-full"></div>
        </div>
        <button
          class="flex items-center text-[15px] justify-center h-[48px] p-1 hover:bg-[#F2F2F2] font-medium rounded-lg mx-1 cursor-pointer"
        >
          Ảnh
        </button>
        <button
          class="flex items-center text-[15px] justify-center h-[48px] p-1 hover:bg-[#F2F2F2] font-medium rounded-lg mx-1 cursor-pointer"
        >
          Video
        </button>
        <button
          class="flex items-center text-[15px] justify-center h-[48px] p-1 hover:bg-[#F2F2F2] font-medium rounded-lg mx-1 cursor-pointer"
        >
          Check in
        </button>
      </div>
    </div>
  </div>
</template>
