<template>
  <Dialog v-model="show" :options="{ size: '3xl' }">
    <template #body>
      <div class="flex flex-col h-full">
        <div class="flex flex-col flex-1 bg-surface-modal">
          <div class="px-4 pb-6 pt-5 sm:px-6">
            <div class="mb-5 flex items-center justify-between">
              <div>
                <h3 class="text-2xl font-semibold leading-6 text-ink-gray-9">
                  {{ __('Create Deal') }}
                </h3>
              </div>
              <div class="flex items-center gap-1">
                <Button
                  v-if="isManager()"
                  variant="ghost"
                  class="w-7"
                  @click="openQuickEntryModal"
                >
                  <EditIcon class="h-4 w-4" />
                </Button>
                <Button variant="ghost" class="w-7" @click="show = false">
                  <FeatherIcon name="x" class="h-4 w-4" />
                </Button>
              </div>
            </div>
          </div>
          <div class="flex-1 overflow-y-auto px-4 sm:px-6 [&_[id^='headlessui-tabs-panel']]:!overflow-visible">
            <div class="mb-4 grid grid-cols-1 gap-4 sm:grid-cols-3">
              <div class="flex items-center gap-3 text-sm text-ink-gray-5">
                <div>{{ __('Choose Existing Organization') }}</div>
                <Switch v-model="chooseExistingOrganization" />
              </div>
              <div class="flex items-center gap-3 text-sm text-ink-gray-5">
                <div>{{ __('Choose Existing Contact') }}</div>
                <Switch v-model="chooseExistingContact" />
              </div>
            </div>
            <div class="h-px w-full border-t my-5" />
            <FieldLayout
              v-if="filteredSections.length"
              :tabs="filteredSections"
              :data="deal"
              :modal="true"
            />
            <ErrorMessage class="mt-4" v-if="error" :message="__(error)" />
          </div>
          <div class="mt-auto px-4 pb-7 pt-4 sm:px-6">
            <div class="flex flex-row-reverse gap-2">
              <Button
                variant="solid"
                :label="__('Create')"
                :loading="isDealCreating"
                @click="createDeal"
              />
            </div>
          </div>
        </div>
      </div>
    </template>
  </Dialog>
</template>

<script setup>
import EditIcon from '@/components/Icons/EditIcon.vue'
import FieldLayout from '@/components/FieldLayout.vue'
import { usersStore } from '@/stores/users'
import { statusesStore } from '@/stores/statuses'
import { capture } from '@/telemetry'
import { Switch, createResource } from 'frappe-ui'
import { computed, ref, reactive, onMounted, nextTick } from 'vue'
import { useRouter } from 'vue-router'
import { translateDealStatus } from '@/utils/dealStatusTranslations'
import { getParsedFields } from '@/utils/getParsedFields'
import { handleDuplicateEntry } from '@/utils/handleDuplicateEntry'

const props = defineProps({
  defaults: Object,
})

const { getUser, isManager } = usersStore()
const { getDealStatus, statusOptions } = statusesStore()

const show = defineModel()
const router = useRouter()
const error = ref(null)

const deal = reactive({
  organization: '',
  organization_name: '',
  website: '',
  no_of_employees: '',
  territory: '',
  annual_revenue: '',
  industry: '',
  contact: '',
  salutation: '',
  first_name: '',
  last_name: '',
  email: '',
  mobile_no: '',
  gender: '',
  status: '',
  deal_owner: '',
})

const isDealCreating = ref(false)
const chooseExistingContact = ref(false)
const chooseExistingOrganization = ref(false)

const tabs = createResource({
  url: 'crm.fcrm.doctype.crm_fields_layout.crm_fields_layout.get_fields_layout',
  cache: ['QuickEntry', 'CRM Deal'],
  params: { doctype: 'CRM Deal', type: 'Quick Entry' },
  auto: true,
  transform: (_tabs) => {
    return _tabs.map(tab => {
      const newTab = {
        ...tab,
        sections: getParsedFields(_tabs, 'CRM Deal', deal, {
          organization: {
            create: (value, close) => {
              _organization.value = { organization_name: value }
              showOrganizationModal.value = true
              close()
            },
            link: (org) => router.push({
              name: 'Organization',
              params: { organizationId: org },
            })
          }
        })
      }

      newTab.sections.forEach((section) => {
        section.fields?.forEach((field) => {
          if (field.name == 'status') {
            field.type = 'Select'
            field.prefixFn = (value) => {
              const statusInfo = getDealStatus(value)
              return statusInfo?.iconColorClass[0]
            }
            field.options = dealStatuses.value.map(status => ({
              label: translateDealStatus(status.value),
              value: status.value
            }))
            field.doctype = 'CRM Deal'
          } else if (field.name == 'deal_owner') {
            field.type = 'Link'
            field.options = 'User'
          }
        })
      })

      return newTab
    })
  },
})

const filteredSections = computed(() => {
  let allSections = tabs.data?.[0]?.sections || []
  if (!allSections.length) return []

  let _filteredSections = []

  if (chooseExistingOrganization.value) {
    _filteredSections.push(
      allSections.find((s) => s.label === 'Select Organization'),
    )
  } else {
    _filteredSections.push(
      allSections.find((s) => s.label === 'Organization Details'),
    )
  }

  if (chooseExistingContact.value) {
    _filteredSections.push(
      allSections.find((s) => s.label === 'Select Contact'),
    )
  } else {
    _filteredSections.push(
      allSections.find((s) => s.label === 'Contact Details'),
    )
  }

  allSections.forEach((s) => {
    if (
      ![
        'Select Organization',
        'Organization Details',
        'Select Contact',
        'Contact Details',
      ].includes(s.label)
    ) {
      _filteredSections.push(s)
    }
  })

  return [{ no_tabs: true, sections: _filteredSections }]
})

const dealStatuses = computed(() => {
  let statuses = statusOptions('deal')
  if (!deal.status) {
    deal.status = statuses[0].value
  }
  return statuses
})

function createDeal() {
  if (deal.website && !deal.website.startsWith('http')) {
    deal.website = 'https://' + deal.website
  }
  createResource({
    url: 'crm.fcrm.doctype.crm_deal.crm_deal.create_deal',
    params: { args: deal },
    auto: true,
    validate() {
      error.value = null
      if (deal.annual_revenue) {
        deal.annual_revenue = deal.annual_revenue.replace(/,/g, '')
        if (isNaN(deal.annual_revenue)) {
          error.value = __('Annual Revenue should be a number')
          return error.value
        }
      }
      if (deal.mobile_no && isNaN(deal.mobile_no.replace(/[-+() ]/g, ''))) {
        error.value = __('Mobile No should be a number')
        return error.value
      }
      if (deal.email && !deal.email.includes('@')) {
        error.value = __('Invalid Email')
        return error.value
      }
      if (!deal.status) {
        error.value = __('Status is required')
        return error.value
      }
      isDealCreating.value = true
    },
    onSuccess(name) {
      capture('deal_created')
      isDealCreating.value = false
      show.value = false
      router.push({ name: 'Deal', params: { dealId: name } })
    },
    async onError(err) {
      // Try to handle duplicate entry error
      const handled = await handleDuplicateEntry(err, 'CRM Deal', () => createDeal())
      if (!handled) {
        isDealCreating.value = false
        if (!err.messages) {
          error.value = err.message
          return
        }
        error.value = err.messages.join('\n')
      }
    },
  })
}

const showQuickEntryModal = defineModel('quickEntry')

function openQuickEntryModal() {
  showQuickEntryModal.value = true
  nextTick(() => {
    show.value = false
  })
}

onMounted(() => {
  Object.assign(deal, props.defaults)
  if (!deal.deal_owner) {
    deal.deal_owner = getUser().name
  }
  if (!deal.status && dealStatuses.value[0].value) {
    deal.status = dealStatuses.value[0].value
  }
})
</script>

<style scoped>
:deep(.flex-col.overflow-y-auto) {
  overflow: visible !important;
}
</style>
