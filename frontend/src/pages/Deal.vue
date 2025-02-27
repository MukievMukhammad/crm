<template>
  <LayoutHeader v-if="deal.data">
    <template #left-header>
      <Breadcrumbs :items="breadcrumbs">
        <template #prefix="{ item }">
          <Icon v-if="item.icon" :icon="item.icon" class="mr-2 h-4" />
        </template>
      </Breadcrumbs>
    </template>
    <template #right-header>
      <CustomActions v-if="customActions" :actions="customActions" />
      <component :is="deal.data._assignedTo?.length == 1 ? 'Button' : 'div'">
        <MultipleAvatar
          :avatars="deal.data._assignedTo"
          @click="showAssignmentModal = true"
        />
      </component>
      <Dropdown :options="statusOptions('deal', updateField, customStatuses)">
        <template #default="{ open }">
          <Button
            :label="translateDealStatus(deal.data.status)"
            :class="getDealStatus(deal.data.status).colorClass"
          >
            <template #prefix>
              <IndicatorIcon />
            </template>
            <template #suffix>
              <FeatherIcon
                :name="open ? 'chevron-up' : 'chevron-down'"
                class="h-4"
              />
            </template>
          </Button>
        </template>
      </Dropdown>
    </template>
  </LayoutHeader>
  <div v-if="deal.data" class="flex h-full overflow-hidden">
    <Tabs v-model="tabIndex" :tabs="tabs" class="!h-full">
      <Activities
        ref="activities"
        doctype="CRM Deal"
        :tabs="tabs"
        v-model:reload="reload"
        v-model:tabIndex="tabIndex"
        v-model="deal"
        :doc="deal"
      />
    </Tabs>
    <Resizer side="right" class="flex flex-col justify-between border-l">
      <div
        class="flex h-10.5 cursor-copy items-center border-b px-5 py-2.5 text-lg font-medium text-ink-gray-9"
        @click="copyToClipboard(deal.data.name)"
      >
        {{ __(deal.data.name) }}
      </div>
      <div class="flex items-center justify-start gap-5 border-b p-5">
        <Tooltip :text="__('Organization logo')">
          <div class="group relative size-12">
            <Avatar
              size="3xl"
              class="size-12"
              :label="displayName"
              :image="organization.data?.organization_logo"
            />
          </div>
        </Tooltip>
        <div class="flex flex-col gap-2.5 truncate text-ink-gray-9">
          <Tooltip :text="displayName">
            <div class="truncate text-2xl font-medium">
              {{ displayName }}
            </div>
          </Tooltip>
          <div class="flex gap-1.5">
            <Tooltip v-if="callEnabled" :text="__('Make a call')">
              <Button class="h-7 w-7" @click="triggerCall">
                <PhoneIcon class="h-4 w-4" />
              </Button>
            </Tooltip>

            <Tooltip :text="__('Call via phone app')">
              <Button
                v-if="primaryContactMobileNo && !callEnabled"
                size="sm"
                @click="trackPhoneActivities('phone')"
              >
                <PhoneIcon class="h-4 w-4" />
              </Button>
            </Tooltip>
            <Tooltip :text="__('Open WhatsApp')">
              <Button
                v-if="primaryContactMobileNo"
                size="sm"
                @click="trackPhoneActivities('whatsapp')"
              >
                <WhatsAppIcon class="h-4 w-4" />
              </Button>
            </Tooltip>
            <Tooltip :text="__('Send an email')">
              <Button class="h-7 w-7">
                <Email2Icon
                  class="h-4 w-4"
                  @click="
                    deal.data.email
                      ? openEmailBox()
                      : errorMessage(__('No email set'))
                  "
                />
              </Button>
            </Tooltip>
            <Tooltip :text="__('Go to website')">
              <Button class="h-7 w-7">
                <LinkIcon
                  class="h-4 w-4"
                  @click="
                    deal.data.website
                      ? openWebsite(deal.data.website)
                      : errorMessage(__('No website set'))
                  "
                />
              </Button>
            </Tooltip>
            <Tooltip :text="__('Attach a file')">
              <Button class="size-7" @click="showFilesUploader = true">
                <AttachmentIcon class="size-4" />
              </Button>
            </Tooltip>
          </div>
        </div>
      </div>
      <SLASection
        v-if="deal.data.sla_status"
        v-model="deal.data"
        @updateField="updateField"
      />
      <div
        v-if="fieldsLayout.data"
        class="flex flex-1 flex-col justify-between overflow-hidden"
      >
        <div class="flex flex-col overflow-y-auto dark-scrollbar">
          <div
            v-for="(section, i) in fieldsLayout.data"
            :key="section.label"
            class="section flex flex-col p-3"
            :class="{ 'border-b': i !== fieldsLayout.data.length - 1 }"
          >
            <Section
              class="px-2 font-semibold"
              :label="section.label"
              :opened="section.opened"
            >
              <template #actions>
                <div v-if="section.contacts" class="pr-2">
                  <Link
                    value=""
                    doctype="Contact"
                    @change="(e) => addContact(e)"
                    :onCreate="
                      (value, close) => {
                        _contact = {
                          first_name: value,
                          company_name: deal.data.organization,
                        }
                        showContactModal = true
                        close()
                      }
                    "
                  >
                    <template #target="{ togglePopover }">
                      <Button
                        class="h-7 px-3"
                        variant="ghost"
                        icon="plus"
                        @click="togglePopover()"
                      />
                    </template>
                  </Link>
                </div>
                <Button
                  v-else-if="
                    ((!section.contacts && i == 1) || i == 0) && isManager()
                  "
                  variant="ghost"
                  class="w-7 mr-2"
                  @click="showSidePanelModal = true"
                >
                  <EditIcon class="h-4 w-4" />
                </Button>
              </template>
              <SidePanelLayout
                v-if="section.fields"
                :fields="section.fields"
                :isLastSection="i == fieldsLayout.data.length - 1"
                v-model="deal.data"
                @update="updateField"
              />
              <div v-else>
                <div
                  v-if="
                    dealContacts?.loading && dealContacts?.data?.length == 0
                  "
                  class="flex min-h-20 flex-1 items-center justify-center gap-3 text-base text-ink-gray-4"
                >
                  <LoadingIndicator class="h-4 w-4" />
                  <span>{{ __('Loading...') }}</span>
                </div>
                <div
                  v-else-if="dealContacts?.data?.length"
                  v-for="(contact, i) in dealContacts.data"
                  :key="contact.name"
                >
                  <div
                    class="px-2 pb-2.5"
                    :class="[i == 0 ? 'pt-5' : 'pt-2.5']"
                  >
                    <Section :opened="contact.opened">
                      <template #header="{ opened, toggle }">
                        <div
                          class="flex cursor-pointer items-center justify-between gap-2 pr-1 text-base leading-5 text-ink-gray-7"
                        >
                          <div
                            class="flex h-7 items-center gap-2 truncate"
                            @click="toggle()"
                          >
                            <Avatar
                              :label="contact.full_name"
                              :image="contact.image"
                              size="md"
                            />
                            <div class="truncate">
                              {{ contact.full_name }}
                            </div>
                            <Badge
                              v-if="contact.is_primary"
                              class="ml-2"
                              variant="outline"
                              :label="__('Primary')"
                              theme="green"
                            />
                          </div>
                          <div class="flex items-center">
                            <Dropdown :options="contactOptions(contact)">
                              <Button
                                icon="more-horizontal"
                                class="text-ink-gray-5"
                                variant="ghost"
                              />
                            </Dropdown>
                            <Button
                              variant="ghost"
                              @click="
                                router.push({
                                  name: 'Contact',
                                  params: { contactId: contact.name },
                                })
                              "
                            >
                              <ArrowUpRightIcon class="h-4 w-4" />
                            </Button>
                            <Button variant="ghost" @click="toggle()">
                              <FeatherIcon
                                name="chevron-right"
                                class="h-4 w-4 text-ink-gray-9 transition-all duration-300 ease-in-out"
                                :class="{ 'rotate-90': opened }"
                              />
                            </Button>
                          </div>
                        </div>
                      </template>
                      <div
                        class="flex flex-col gap-1.5 text-base text-ink-gray-8"
                      >
                        <div class="flex items-center gap-3 pb-1.5 pl-1 pt-4">
                          <Email2Icon class="h-4 w-4" />
                          {{ contact.email }}
                        </div>
                        <div class="flex items-center gap-3 p-1 py-1.5">
                          <PhoneIcon class="h-4 w-4" />
                          {{ contact.mobile_no }}
                        </div>
                      </div>
                    </Section>
                  </div>
                  <div
                    v-if="i != dealContacts.data.length - 1"
                    class="mx-2 h-px border-t border-outline-gray-modals"
                  />
                </div>
                <div
                  v-else
                  class="flex h-20 items-center justify-center text-base text-ink-gray-5"
                >
                  {{ __('No contacts added') }}
                </div>
              </div>
            </Section>
          </div>
        </div>
      </div>
    </Resizer>
  </div>
  <OrganizationModal
    v-if="showOrganizationModal"
    v-model="showOrganizationModal"
    v-model:organization="_organization"
    :options="{
      redirect: false,
      afterInsert: (doc) => updateField('organization', doc.name)
    }"
  />
  <ContactModal
    v-model="showContactModal"
    :contact="_contact"
    :options="{
      redirect: false,
      afterInsert: (doc) => addContact(doc.name),
    }"
  />
  <AssignmentModal
    v-if="showAssignmentModal"
    v-model="showAssignmentModal"
    v-model:assignees="deal.data._assignedTo"
    :doc="deal.data"
    doctype="CRM Deal"
  />
  <SidePanelModal
    v-if="showSidePanelModal"
    v-model="showSidePanelModal"
    doctype="CRM Deal"
    @reload="() => fieldsLayout.reload()"
  />
  <FilesUploader
    v-if="deal.data?.name"
    v-model="showFilesUploader"
    doctype="CRM Deal"
    :docname="deal.data.name"
    @after="
      () => {
        activities?.all_activities?.reload()
        changeTabTo('attachments')
      }
    "
  />
  <QuickEntryModal
    v-if="showQuickEntryModal"
    v-model="showQuickEntryModal"
    doctype="CRM Organization"
    :options="{
      redirect: false,
      afterInsert: (doc) => updateField('organization', doc.name),
    }"
  />
</template>
<script setup>
import Icon from '@/components/Icon.vue'
import Resizer from '@/components/Resizer.vue'
import LoadingIndicator from '@/components/Icons/LoadingIndicator.vue'
import EditIcon from '@/components/Icons/EditIcon.vue'
import ActivityIcon from '@/components/Icons/ActivityIcon.vue'
import EmailIcon from '@/components/Icons/EmailIcon.vue'
import Email2Icon from '@/components/Icons/Email2Icon.vue'
import CommentIcon from '@/components/Icons/CommentIcon.vue'
import DetailsIcon from '@/components/Icons/DetailsIcon.vue'
import PhoneIcon from '@/components/Icons/PhoneIcon.vue'
import TaskIcon from '@/components/Icons/TaskIcon.vue'
import NoteIcon from '@/components/Icons/NoteIcon.vue'
import WhatsAppIcon from '@/components/Icons/WhatsAppIcon.vue'
import IndicatorIcon from '@/components/Icons/IndicatorIcon.vue'
import LinkIcon from '@/components/Icons/LinkIcon.vue'
import ArrowUpRightIcon from '@/components/Icons/ArrowUpRightIcon.vue'
import SuccessIcon from '@/components/Icons/SuccessIcon.vue'
import AttachmentIcon from '@/components/Icons/AttachmentIcon.vue'
import LayoutHeader from '@/components/LayoutHeader.vue'
import Activities from '@/components/Activities/Activities.vue'
import OrganizationModal from '@/components/Modals/OrganizationModal.vue'
import AssignmentModal from '@/components/Modals/AssignmentModal.vue'
import FilesUploader from '@/components/FilesUploader/FilesUploader.vue'
import MultipleAvatar from '@/components/MultipleAvatar.vue'
import ContactModal from '@/components/Modals/ContactModal.vue'
import SidePanelModal from '@/components/Modals/SidePanelModal.vue'
import Link from '@/components/Controls/Link.vue'
import Section from '@/components/Section.vue'
import SidePanelLayout from '@/components/SidePanelLayout.vue'
import SLASection from '@/components/SLASection.vue'
import CustomActions from '@/components/CustomActions.vue'
import QuickEntryModal from '@/components/Modals/QuickEntryModal.vue'
import {
  openWebsite,
  createToast,
  setupAssignees,
  setupCustomizations,
  errorMessage,
  copyToClipboard,
} from '@/utils'
import { getView } from '@/utils/view'
import { globalStore } from '@/stores/global'
import { statusesStore } from '@/stores/statuses'
import { usersStore } from '@/stores/users'
import { whatsappEnabled, callEnabled } from '@/composables/settings'
import {
  createResource,
  Dropdown,
  Tooltip,
  Avatar,
  Tabs,
  Breadcrumbs,
  call,
  usePageMeta,
  createDocumentResource,
} from 'frappe-ui'
import { ref, computed, h, onMounted, onBeforeUnmount } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { useActiveTabManager } from '@/composables/useActiveTabManager'
import { trackCommunication } from '@/utils/communicationUtils'
import { translateDealStatus } from '@/utils/dealStatusTranslations'
import { getParsedFields } from '@/utils/getParsedFields'

const { $dialog, $socket, makeCall } = globalStore()
const { statusOptions, getDealStatus } = statusesStore()
const { isManager } = usersStore()
const route = useRoute()
const router = useRouter()

const props = defineProps({
  dealId: {
    type: String,
    required: true,
  },
})

const customActions = ref([])
const customStatuses = ref([])

const deal = createResource({
  url: 'crm.fcrm.doctype.crm_deal.api.get_deal',
  params: { name: props.dealId },
  cache: ['deal', props.dealId],
  onSuccess: async (data) => {
    if (data.organization) {
      organization.update({
        params: { doctype: 'CRM Organization', name: data.organization },
      })
      organization.fetch()
    }

    let obj = {
      doc: data,
      $dialog,
      $socket,
      router,
      updateField,
      createToast,
      deleteDoc: deleteDeal,
      resource: {
        deal,
        dealContacts,
        fieldsLayout,
      },
      call,
    }
    setupAssignees(data)
    let customization = await setupCustomizations(data, obj)
    customActions.value = customization.actions || []
    customStatuses.value = customization.statuses || []
  },
})

const organization = createResource({
  url: 'frappe.client.get',
  onSuccess: (data) => (deal.data._organizationObj = data),
})

const dealContacts = createResource({
  url: 'crm.fcrm.doctype.crm_deal.api.get_deal_contacts',
  params: { name: props.dealId },
  cache: ['deal_contacts', props.dealId],
  auto: true,
  transform: (data) => {
    data.forEach((contact) => {
      contact.opened = false
    })
    return data
  },
})

onMounted(() => {
  $socket.on('crm_customer_created', () => {
    createToast({
      title: __('Customer created successfully'),
      icon: 'check',
      iconClasses: 'text-ink-green-3',
    })
  })

  if (deal.data) {
    organization.data = deal.data._organizationObj
    return
  }
  deal.fetch()
})

onBeforeUnmount(() => {
  $socket.off('crm_customer_created')
})

const reload = ref(false)
const showOrganizationModal = ref(false)
const showAssignmentModal = ref(false)
const showSidePanelModal = ref(false)
const showFilesUploader = ref(false)
const showQuickEntryModal = ref(false)
const _organization = ref({})
const _contact = ref({})

function updateField(name, value, callback) {
  if (validateRequired(name, value)) return
  
  updateDeal(name, value, () => {
    deal.data[name] = value
    callback?.()
  })
}

function validateRequired(fieldname, value) {
  let meta = deal.data.fields_meta || {}
  if (meta[fieldname]?.reqd && !value) {
    createToast({
      title: __('Error Updating Deal'),
      text: __('{0} is a required field', [meta[fieldname].label]),
      icon: 'x',
      iconClasses: 'text-ink-red-4',
    })
    return true
  }
  return false
}

function updateDeal(fieldname, value, callback) {
  value = Array.isArray(fieldname) ? '' : value

  createResource({
    url: 'frappe.client.set_value',
    params: {
      doctype: 'CRM Deal',
      name: props.dealId,
      fieldname,
      value,
    },
    auto: true,
    onSuccess: () => {
      deal.reload()
      reload.value = true
      createToast({
        title: __('Deal updated'),
        icon: 'check',
        iconClasses: 'text-ink-green-3',
      })
      callback?.()
    },
    onError: (err) => {
      createToast({
        title: __('Error updating deal'),
        text: __(err.messages?.[0]),
        icon: 'x',
        iconClasses: 'text-ink-red-4',
      })
    },
  })
}

const displayName = computed(() => {
  if (!deal.data) return __('Loading...')
  
  if (organization.data?.name) {
    return organization.data.name
  }
  
  if (dealContacts.data) {
    const primaryContact = dealContacts.data.find(c => c.is_primary)
    if (primaryContact?.full_name) {
      return primaryContact.full_name
    }
  }
  
  return __('Untitled')
})

const breadcrumbs = computed(() => {
  let items = [{ label: __('Deals'), route: { name: 'Deals' } }]

  if (route.query.view || route.query.viewType) {
    let view = getView(route.query.view, route.query.viewType, 'CRM Deal')
    if (view) {
      items.push({
        label: __(view.label),
        icon: view.icon,
        route: {
          name: 'Deals',
          params: { viewType: route.query.viewType },
          query: { view: route.query.view },
        },
      })
    }
  }

  items.push({
    label: displayName.value,
    route: { name: 'Deal', params: { dealId: deal.data.name } },
  })
  return items
})

usePageMeta(() => ({
  title: displayName.value,
}))

const tabs = computed(() => {
  let tabOptions = [
    {
      name: 'Activity',
      label: __('Activity'),
      icon: ActivityIcon,
    },
    {
      name: 'Emails',
      label: __('Emails'),
      icon: EmailIcon,
    },
    {
      name: 'Comments',
      label: __('Comments'),
      icon: CommentIcon,
    },
    {
      name: 'Data',
      label: __('Data'),
      icon: DetailsIcon,
    },
    {
      name: 'Calls',
      label: __('Calls'),
      icon: PhoneIcon,
      condition: () => callEnabled.value,
    },
    {
      name: 'Tasks',
      label: __('Tasks'),
      icon: TaskIcon,
    },
    {
      name: 'Notes',
      label: __('Notes'),
      icon: NoteIcon,
    },
    {
      name: 'Attachments',
      label: __('Attachments'),
      icon: AttachmentIcon,
    },
    {
      name: 'WhatsApp',
      label: __('WhatsApp'),
      icon: WhatsAppIcon,
      condition: () => whatsappEnabled.value,
    },
  ]
  return tabOptions.filter((tab) => (tab.condition ? tab.condition() : true))
})
const { tabIndex } = useActiveTabManager(tabs, 'lastDealTab')

const fieldsLayout = createResource({
  url: 'crm.api.doc.get_sidebar_fields',
  cache: ['fieldsLayout', props.dealId],
  params: { doctype: 'CRM Deal', name: props.dealId },
  auto: true,
  transform: (data) => getParsedFields(data, 'CRM Deal', deal.data, {
    organization: {
      create: (value, close) => {
        _organization.value = { organization_name: value }
        showOrganizationModal.value = true
        close()
      },
      link: (org) => router.push({
        name: 'Organization',
        params: { organizationId: org },
      }),
      edit: async (org) => {
        showQuickEntryModal.value = true
      }
    },
    contact: {
      link: (contact) => router.push({
        name: 'Contact',
        params: { contactId: contact },
      }),
      onChange: (contact) => addContact(contact)
    }
  })
})

const showContactModal = ref(false)

function contactOptions(contact) {
  let options = []

  if (!contact.is_primary) {
    options.push({
      label: __('Set as Primary Contact'),
      icon: h(SuccessIcon, { class: 'h-4 w-4' }),
      onClick: () => setPrimaryContact(contact.name),
    })
  }

  return options
}

async function addContact(contact) {
  let d = await call('crm.fcrm.doctype.crm_deal.crm_deal.add_contact', {
    deal: props.dealId,
    contact,
  })
  if (d) {
    dealContacts.reload()
    fieldsLayout.reload()
    createToast({
      title: __('Contact added'),
      icon: 'check',
      iconClasses: 'text-ink-green-3',
    })
  }
}

async function setPrimaryContact(contact) {
  let d = await call('crm.fcrm.doctype.crm_deal.crm_deal.set_primary_contact', {
    deal: props.dealId,
    contact,
  })
  if (d) {
    dealContacts.reload()
    createToast({
      title: __('Primary contact set'),
      icon: 'check',
      iconClasses: 'text-ink-green-3',
    })
  }
}

function trackPhoneActivities(type = 'phone') {
  const primaryContact = dealContacts.data?.find(c => c.is_primary)
  if (!primaryContact?.mobile_no) {
    errorMessage(__('No phone number set'))
    return
  }
  trackCommunication({
    type,
    doctype: 'CRM Deal',
    docname: deal.data.name,
    phoneNumber: primaryContact.mobile_no,
    activities: activities.value,
    contactName: primaryContact.name
  })
}

function triggerCall() {
  let primaryContact = dealContacts.data?.find((c) => c.is_primary)
  let mobile_no = primaryContact.mobile_no || null

  if (!primaryContact) {
    errorMessage(__('No primary contact set'))
    return
  }

  if (!mobile_no) {
    errorMessage(__('No mobile number set'))
    return
  }

  makeCall(mobile_no)
}

async function deleteDeal(name) {
  await call('frappe.client.delete', {
    doctype: 'CRM Deal',
    name,
  })
  router.push({ name: 'Deals' })
}

const activities = ref(null)

function openEmailBox() {
  activities.value.emailBox.show = true
}

const primaryContactMobileNo = computed(() => {
  return dealContacts.data?.find(c => c.is_primary)?.mobile_no
})
</script>

<style scoped>
:deep(.section:has(.section-field.hidden)) {
  display: none;
}
:deep(.section:has(.section-field:not(.hidden))) {
  display: flex;
}
</style>
