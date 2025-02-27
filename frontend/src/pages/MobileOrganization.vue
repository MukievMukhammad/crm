<template>
  <LayoutHeader v-if="organization.doc">
    <header
      class="relative flex h-10.5 items-center justify-between gap-2 py-2.5 pl-2"
    >
      <Breadcrumbs :items="breadcrumbs">
        <template #prefix="{ item }">
          <Icon v-if="item.icon" :icon="item.icon" class="mr-2 h-4" />
        </template>
      </Breadcrumbs>
    </header>
  </LayoutHeader>
  <div v-if="organization.doc" class="flex flex-col h-full overflow-hidden">
    <FileUploader
      @success="changeOrganizationImage"
      :validateFile="validateFile"
    >
      <template #default="{ openFileSelector, error }">
        <div class="flex flex-col items-start justify-start gap-4 p-4">
          <div class="flex gap-4 items-center">
            <div class="group relative h-14.5 w-14.5">
              <Avatar
                size="3xl"
                class="h-14.5 w-14.5"
                :label="organization.doc.organization_name"
                :image="organization.doc.organization_logo"
              />
              <component
                :is="organization.doc.organization_logo ? Dropdown : 'div'"
                v-bind="
                  organization.doc.organization_logo
                    ? {
                        options: [
                          {
                            icon: 'upload',
                            label: organization.doc.organization_logo
                              ? __('Change image')
                              : __('Upload image'),
                            onClick: openFileSelector,
                          },
                          {
                            icon: 'trash-2',
                            label: __('Remove image'),
                            onClick: () => changeOrganizationImage(''),
                          },
                        ],
                      }
                    : { onClick: openFileSelector }
                "
                class="!absolute bottom-0 left-0 right-0"
              >
                <div
                  class="z-1 absolute bottom-0 left-0 right-0 flex h-14 cursor-pointer items-center justify-center rounded-b-full bg-black bg-opacity-40 pt-5 opacity-0 duration-300 ease-in-out group-hover:opacity-100"
                  style="
                    -webkit-clip-path: inset(22px 0 0 0);
                    clip-path: inset(22px 0 0 0);
                  "
                >
                  <CameraIcon class="h-6 w-6 cursor-pointer text-white" />
                </div>
              </component>
            </div>
            <div class="flex flex-col gap-2 truncate">
              <div class="truncate text-lg font-medium text-ink-gray-9">
                {{ organization.doc.name }}
              </div>
              <div class="flex items-center gap-1.5">
                <Button
                  size="sm"
                  class="dark:text-white dark:hover:bg-gray-700"
                  @click="openWebsite"
                >
                  <template #prefix>
                    <FeatherIcon name="link" class="h-4 w-4" />
                  </template>
                  {{ __('Open website') }}
                </Button>
                <Button
                  :label="__('Delete')"
                  variant="ghost"
                  theme="red"
                  size="sm"
                  class="dark:text-red-400 dark:hover:bg-gray-700"
                  @click="deleteOrganization"
                >
                  <template #prefix>
                    <FeatherIcon name="trash-2" class="h-4 w-4" />
                  </template>
                </Button>
              </div>
              <ErrorMessage :message="__(error)" />
            </div>
          </div>
        </div>
      </template>
    </FileUploader>
    <Tabs
      v-model="tabIndex"
      :tabs="tabs"
      tablistClass="!px-4"
      class="overflow-auto"
    >
      <template #tab="{ tab, selected }">
        <button
          v-if="tab.name !== 'Details'"
          class="group flex items-center gap-2 border-b border-transparent py-2.5 text-base text-ink-gray-5 duration-300 ease-in-out hover:border-outline-gray-3 hover:text-ink-gray-9"
          :class="{ 'text-ink-gray-9': selected }"
        >
          <component v-if="tab.icon" :is="tab.icon" class="h-5" />
          {{ __(tab.label) }}
          <Badge
            class="group-hover:bg-surface-gray-7"
            :class="[selected ? 'bg-surface-gray-7' : 'bg-gray-600']"
            variant="solid"
            theme="gray"
            size="sm"
          >
            {{ tab.count }}
          </Badge>
        </button>
      </template>
      <template #default="{ tab }">
        <div class="flex-1 relative">
          <div v-if="tab.name === 'Details'">
            <div
              v-if="fieldsLayout.data"
              class="flex flex-1 flex-col justify-between overflow-hidden"
            >
              <div class="flex flex-col overflow-y-auto">
                <div
                  v-for="(section, i) in fieldsLayout.data"
                  :key="section.label"
                  class="flex flex-col px-2 py-3 sm:p-3"
                  :class="{ 'border-b': i !== fieldsLayout.data.length - 1 }"
                >
                  <Section :label="section.label" :opened="section.opened">
                    <SidePanelLayout
                      :fields="section.fields"
                      :isLastSection="i == fieldsLayout.data.length - 1"
                      v-model="organization.doc"
                      @update="updateField"
                    />
                  </Section>
                </div>
              </div>
            </div>
          </div>
          <div 
            v-else
            class="absolute inset-0 overflow-auto"
          >
            <div class="min-w-max h-full">
              <DealsListView
                v-if="tab.name === 'Deals' && rows.length"
                class="mt-4"
                :rows="rows"
                :columns="columns"
                :options="{ selectable: false, showTooltip: false }"
              />
              <ContactsListView
                v-if="tab.name === 'Contacts' && rows.length"
                class="mt-4"
                :rows="rows"
                :columns="columns"
                :options="{ selectable: false, showTooltip: false }"
              />
              <div
                v-if="!rows.length"
                class="h-[calc(100vh-200px)] flex items-center justify-center text-xl font-medium text-ink-gray-4"
              >
                <div class="flex flex-col items-center justify-center space-y-3">
                  <component :is="tab.icon" class="!h-10 !w-10" />
                  <div>{{ __('Not Found') }}</div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </template>
    </Tabs>
  </div>
  <AddressModal 
    v-model="showAddressModal" 
    v-model:address="_address"
    @save="updateField('address', $event)"
  />
</template>

<script setup>
import Section from '@/components/Section.vue'
import SidePanelLayout from '@/components/SidePanelLayout.vue'
import Icon from '@/components/Icon.vue'
import LayoutHeader from '@/components/LayoutHeader.vue'
import AddressModal from '@/components/Modals/AddressModal.vue'
import DealsListView from '@/components/ListViews/DealsListView.vue'
import ContactsListView from '@/components/ListViews/ContactsListView.vue'
import DetailsIcon from '@/components/Icons/DetailsIcon.vue'
import CameraIcon from '@/components/Icons/CameraIcon.vue'
import DealsIcon from '@/components/Icons/DealsIcon.vue'
import ContactsIcon from '@/components/Icons/ContactsIcon.vue'
import { globalStore } from '@/stores/global'
import { usersStore } from '@/stores/users'
import { statusesStore } from '@/stores/statuses'
import { getView } from '@/utils/view'
import {
  formatDate,
  timeAgo,
  formatNumberIntoCurrency,
  createToast,
} from '@/utils'
import {
  Breadcrumbs,
  Avatar,
  FileUploader,
  Dropdown,
  Tabs,
  call,
  createListResource,
  createDocumentResource,
  usePageMeta,
  createResource,
} from 'frappe-ui'
import { h, computed, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'

const props = defineProps({
  organizationId: {
    type: String,
    required: true,
  },
})

const { getUser } = usersStore()
const { $dialog } = globalStore()
const { getDealStatus } = statusesStore()

const route = useRoute()
const router = useRouter()

const organization = createDocumentResource({
  doctype: 'CRM Organization',
  name: props.organizationId,
  cache: ['organization', props.organizationId],
  fields: ['*'],
  auto: true,
})

async function updateField(fieldname, value) {
  if (fieldname === 'organization_name') {
    // Handle renaming
    const newName = await call('frappe.client.rename_doc', {
      doctype: 'CRM Organization',
      old_name: organization.doc.organization_name,
      new_name: value,
    })
    router.push({
      name: 'Organization',
      params: { organizationId: newName }
    })
  } else {
    await call('frappe.client.set_value', {
      doctype: 'CRM Organization',
      name: props.organizationId,
      fieldname: fieldname,
      value: value
    })
    organization.reload()
  }
  
  createToast({
    title: __('Organization updated'),
    icon: 'check',
    iconClasses: 'text-ink-green-3',
  })
}

const breadcrumbs = computed(() => {
  let items = [{ label: __('Organizations'), route: { name: 'Organizations' } }]

  if (route.query.view || route.query.viewType) {
    let view = getView(
      route.query.view,
      route.query.viewType,
      'CRM Organization',
    )
    if (view) {
      items.push({
        label: __(view.label),
        icon: view.icon,
        route: {
          name: 'Organizations',
          params: { viewType: route.query.viewType },
          query: { view: route.query.view },
        },
      })
    }
  }

  items.push({
    label: props.organizationId,
    route: {
      name: 'Organization',
      params: { organizationId: props.organizationId },
    },
  })
  return items
})

usePageMeta(() => {
  return {
    title: props.organizationId,
  }
})

function validateFile(file) {
  let extn = file.name.split('.').pop().toLowerCase()
  if (!['png', 'jpg', 'jpeg'].includes(extn)) {
    return __('Only PNG and JPG images are allowed')
  }
}

async function changeOrganizationImage(file) {
  await call('frappe.client.set_value', {
    doctype: 'CRM Organization',
    name: props.organizationId,
    fieldname: 'organization_logo',
    value: file?.file_url || '',
  })
  organization.reload()
}

async function deleteOrganization() {
  $dialog({
    title: __('Delete organization'),
    message: __('Are you sure you want to delete this organization?'),
    actions: [
      {
        label: __('Delete'),
        theme: 'red',
        variant: 'solid',
        async onClick(close) {
          await call('frappe.client.delete', {
            doctype: 'CRM Organization',
            name: props.organizationId,
          })
          close()
          router.push({ name: 'Organizations' })
        },
      },
    ],
  })
}

function openWebsite() {
  if (!organization.doc.website)
    createToast({
      title: __('Website not found'),
      icon: 'x',
      iconClasses: 'text-ink-red-4',
    })
  else window.open(organization.doc.website, '_blank')
}

const showAddressModal = ref(false)
const _organization = ref({})
const _address = ref({})

const fieldsLayout = createResource({
  url: 'crm.api.doc.get_sidebar_fields',
  cache: ['fieldsLayout', props.organizationId],
  params: { doctype: 'CRM Organization', name: props.organizationId },
  auto: true,
  transform: (data) => getParsedFields(data),
})

function getParsedFields(data) {
  return data.map((section) => {
    return {
      ...section,
      fields: computed(() =>
        section.fields.map((field) => {
          // Get translated field label
          const translatedLabel = __(field.label || field.name.split('_').map(word => 
            word.charAt(0).toUpperCase() + word.slice(1)).join(' '))

          // Handle link fields
          if (field.type === 'link') {
            const baseField = {
              ...field,
              doctype: field.options || field.doctype,
              options: field.options,
              placeholder: `${__('Select')} ${translatedLabel}`
            }

            // Special handling for address field
            if (field.name === 'address') {
              return {
                ...baseField,
                doctype: 'Address',
                create: (value, close) => {
                  _address.value = { address_title: value }
                  showAddressModal.value = true
                  close()
                },
                edit: async (addr) => {
                  _address.value = await call('frappe.client.get', {
                    doctype: 'Address',
                    name: addr,
                  })
                  showAddressModal.value = true
                }
              }
            }

            return baseField
          }

          // Handle select fields
          if (field.type === 'select') {
            return {
              ...field,
              placeholder: `${__('Select')} ${translatedLabel}`
            }
          }

          // Default field handling
          return {
            ...field,
            placeholder: `${__('Enter')} ${translatedLabel}`
          }
        }),
      ),
    }
  })
}

const tabIndex = ref(0)
const tabs = [
  {
    name: 'Details',
    label: __('Details'),
    icon: DetailsIcon,
  },
  {
    name: 'Deals',
    label: __('Deals'),
    icon: h(DealsIcon, { class: 'h-4 w-4' }),
    count: computed(() => deals.data?.length),
  },
  {
    name: 'Contacts',
    label: __('Contacts'),
    icon: h(ContactsIcon, { class: 'h-4 w-4' }),
    count: computed(() => contacts.data?.length),
  },
]

const deals = createListResource({
  type: 'list',
  doctype: 'CRM Deal',
  cache: ['deals', props.organizationId],
  fields: [
    'name',
    'organization',
    'currency',
    'annual_revenue',
    'status',
    'email',
    'mobile_no',
    'deal_owner',
    'modified',
  ],
  filters: {
    organization: props.organizationId,
  },
  orderBy: 'modified desc',
  pageLength: 20,
  auto: true,
})

const contacts = createListResource({
  type: 'list',
  doctype: 'Contact',
  cache: ['contacts', props.organizationId],
  fields: [
    'name',
    'full_name',
    'image',
    'email_id',
    'mobile_no',
    'company_name',
    'modified',
  ],
  filters: [
    ['company_name', 'like', `%${props.organizationId}%`]
  ],
  orderBy: 'modified desc',
  pageLength: 20,
  auto: true,
})

const rows = computed(() => {
  if (tabIndex.value === 0) return [] // Details tab
  
  const currentTab = tabs[tabIndex.value]
  if (currentTab.name === 'Deals') {
    if (!deals.data) return []
    return deals.data.map(getDealRowObject)
  } else if (currentTab.name === 'Contacts') {
    if (!contacts.data) return []
    return contacts.data.map(getContactRowObject)
  }
  return []
})

const columns = computed(() => {
  const currentTab = tabs[tabIndex.value]
  return currentTab.name === 'Deals' ? dealColumns : contactColumns
})

function getDealRowObject(deal) {
  return {
    name: deal.name,
    organization: {
      label: deal.organization,
      logo: props.organization?.organization_logo,
    },
    annual_revenue: formatNumberIntoCurrency(
      deal.annual_revenue,
      deal.currency,
    ),
    status: {
      label: deal.status,
      color: getDealStatus(deal.status)?.iconColorClass,
    },
    email: deal.email,
    mobile_no: deal.mobile_no,
    deal_owner: {
      label: deal.deal_owner && getUser(deal.deal_owner).full_name,
      ...(deal.deal_owner && getUser(deal.deal_owner)),
    },
    modified: {
      label: formatDate(deal.modified),
      timeAgo: __(timeAgo(deal.modified)),
    },
  }
}

function getContactRowObject(contact) {
  return {
    name: contact.name,
    full_name: {
      label: contact.full_name,
      image_label: contact.full_name,
      image: contact.image,
    },
    email: contact.email_id,
    mobile_no: contact.mobile_no,
    company_name: {
      label: contact.company_name,
      logo: props.organization?.organization_logo,
    },
    modified: {
      label: formatDate(contact.modified),
      timeAgo: __(timeAgo(contact.modified)),
    },
  }
}

const dealColumns = [
  {
    label: __('Organization'),
    key: 'organization',
    width: '11rem',
  },
  {
    label: __('Amount'),
    key: 'annual_revenue',
    width: '9rem',
  },
  {
    label: __('Status'),
    key: 'status',
    width: '10rem',
  },
  {
    label: __('Email'),
    key: 'email',
    width: '12rem',
  },
  {
    label: __('Mobile no'),
    key: 'mobile_no',
    width: '11rem',
  },
  {
    label: __('Deal owner'),
    key: 'deal_owner',
    width: '10rem',
  },
  {
    label: __('Last modified'),
    key: 'modified',
    width: '8rem',
  },
]

const contactColumns = [
  {
    label: __('Name'),
    key: 'full_name',
    width: '17rem',
  },
  {
    label: __('Email'),
    key: 'email',
    width: '12rem',
  },
  {
    label: __('Phone'),
    key: 'mobile_no',
    width: '12rem',
  },
  {
    label: __('Organization'),
    key: 'company_name',
    width: '12rem',
  },
  {
    label: __('Last modified'),
    key: 'modified',
    width: '8rem',
  },
]
</script>
