<template>
  <div
    v-if="title !== 'Data'"
    class="mx-4 my-3 flex items-center gap-4 justify-between text-lg font-medium sm:mx-10 sm:mb-4 sm:mt-8"
  >
    <div class="flex h-8 items-center text-xl font-semibold text-ink-gray-8">
      {{ __(title) }}
    </div>
    <Button
      v-if="title == 'Emails'"
      variant="solid"
      class="shrink-0 px-3"
      @click="emailBox.show = true"
    >
      <template #prefix>
        <FeatherIcon name="plus" class="h-4 w-4" />
      </template>
      <span class="btn-text-standard">{{ __('New Email') }}</span>
    </Button>
    <Button
      v-else-if="title == 'Comments'"
      variant="solid"
      @click="emailBox.showComment = true"
    >
      <template #prefix>
        <FeatherIcon name="plus" class="h-4 w-4" />
      </template>
      <span class="btn-text-standard">{{ __('New Comment') }}</span>
    </Button>
    <Button
      v-else-if="title == 'Calls'"
      variant="solid"
      class="shrink-0"
      @click="makeCall(doc.data.mobile_no)"
    >
      <template #prefix>
        <PhoneIcon class="h-4 w-4" />
      </template>
      <span class="btn-text-standard">{{ __('Make a Call') }}</span>
    </Button>
    <Button
      v-else-if="title == 'Notes'"
      variant="solid"
      class="shrink-0"
      @click="modalRef.showNote()"
    >
      <template #prefix>
        <FeatherIcon name="plus" class="h-4 w-4" />
      </template>
      <span class="btn-text-standard">{{ __('New Note') }}</span>
    </Button>
    <Button
      v-else-if="title == 'Tasks'"
      variant="solid"
      class="shrink-0"
      @click="modalRef.showTask()"
    >
      <template #prefix>
        <FeatherIcon name="plus" class="h-4 w-4" />
      </template>
      <span class="btn-text-standard">{{ __('New Task') }}</span>
    </Button>
    <Button
      v-else-if="title == 'Attachments'"
      variant="solid"
      class="shrink-0"
      @click="showFilesUploader = true"
    >
      <template #prefix>
        <FeatherIcon name="plus" class="h-4 w-4" />
      </template>
      <span class="btn-text-standard">{{ __('Upload Attachment') }}</span>
    </Button>
    <div class="flex gap-2 shrink-0" v-else-if="title == 'WhatsApp'">
      <Button
        class="shrink-0"
        @click="showWhatsappTemplates = true"
      >
        <template #prefix>
          <FeatherIcon name="plus" class="h-4 w-4" />
        </template>
        <span class="btn-text-standard">{{ __('Send Template') }}</span>
      </Button>
      <Button variant="solid" class="shrink-0" @click="whatsappBox.show()">
        <template #prefix>
          <FeatherIcon name="plus" class="h-4 w-4" />
        </template>
        <span class="btn-text-standard">{{ __('New Message') }}</span>
      </Button>
    </div>
    <div class="flex gap-2 shrink-0" v-else-if="title == 'Avito'">
      <Button
        :label="__('Send Template')"
        @click="showAvitoTemplates = true"
      />
      <Button variant="solid" @click="avitoBox.show()">
        <template #prefix>
          <FeatherIcon name="plus" class="h-4 w-4" />
        </template>
        <span>{{ __('New Message') }}</span>
      </Button>
    </div>
    <Dropdown v-else :options="defaultActions" @click.stop>
      <template v-slot="{ open }">
        <Button variant="solid" class="shrink-0">
          <template #prefix>
            <FeatherIcon name="plus" class="h-4 w-4" />
          </template>
          <span class="btn-text-standard">{{ __('New') }}</span>
          <template #suffix>
            <FeatherIcon
              :name="open ? 'chevron-up' : 'chevron-down'"
              class="h-4 w-4 hidden sm:inline"
            />
          </template>
        </Button>
      </template>
    </Dropdown>
  </div>
</template>
<script setup>
import Email2Icon from '@/components/Icons/Email2Icon.vue'
import CommentIcon from '@/components/Icons/CommentIcon.vue'
import PhoneIcon from '@/components/Icons/PhoneIcon.vue'
import NoteIcon from '@/components/Icons/NoteIcon.vue'
import TaskIcon from '@/components/Icons/TaskIcon.vue'
import AttachmentIcon from '@/components/Icons/AttachmentIcon.vue'
import WhatsAppIcon from '@/components/Icons/WhatsAppIcon.vue'
import AvitoIcon from '@/components/Icons/AvitoIcon.vue'
import { globalStore } from '@/stores/global'
import { whatsappEnabled, callEnabled } from '@/composables/settings'
import { avitoEnabled } from '@/composables/avito'
import { Dropdown } from 'frappe-ui'
import { computed, h } from 'vue'

const props = defineProps({
  tabs: Array,
  title: String,
  doc: Object,
  modalRef: Object,
  emailBox: Object,
  whatsappBox: Object,
  avitoBox: Object,
})

const { makeCall } = globalStore()

const tabIndex = defineModel()
const showWhatsappTemplates = defineModel('showWhatsappTemplates')
const showAvitoTemplates = defineModel('showAvitoTemplates')
const showFilesUploader = defineModel('showFilesUploader')

const defaultActions = computed(() => {
  let actions = [
    {
      icon: h(Email2Icon, { class: 'h-4 w-4' }),
      label: __('New Email'),
      onClick: () => (props.emailBox.show = true),
    },
    {
      icon: h(CommentIcon, { class: 'h-4 w-4' }),
      label: __('New Comment'),
      onClick: () => (props.emailBox.showComment = true),
    },
    {
      icon: h(PhoneIcon, { class: 'h-4 w-4' }),
      label: __('Make a Call'),
      onClick: () => makeCall(props.doc.data.mobile_no),
      condition: () => callEnabled.value,
    },
    {
      icon: h(NoteIcon, { class: 'h-4 w-4' }),
      label: __('New Note'),
      onClick: () => props.modalRef.showNote(),
    },
    {
      icon: h(TaskIcon, { class: 'h-4 w-4' }),
      label: __('New Task'),
      onClick: () => props.modalRef.showTask(),
    },
    {
      icon: h(AttachmentIcon, { class: 'h-4 w-4' }),
      label: __('Upload Attachment'),
      onClick: () => (showFilesUploader.value = true),
    },
    {
      icon: h(WhatsAppIcon, { class: 'h-4 w-4' }),
      label: __('New WhatsApp Message'),
      onClick: () => (tabIndex.value = getTabIndex('WhatsApp')),
      condition: () => whatsappEnabled.value,
    },
    {
      icon: h(AvitoIcon, { class: 'h-4 w-4' }),
      label: __('New Avito Message'),
      onClick: () => (tabIndex.value = getTabIndex('Avito')),
      condition: () => avitoEnabled.value,
    },
  ]
  return actions.filter((action) =>
    action.condition ? action.condition() : true,
  )
})

function getTabIndex(name) {
  return props.tabs.findIndex((tab) => tab.name === name)
}
</script>
