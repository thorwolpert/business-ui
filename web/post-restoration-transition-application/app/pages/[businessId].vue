<script setup lang="ts">
import { h, resolveComponent } from 'vue'
import { fromIsoToUsDateFormat } from '~/utils/uidate'
import { areUiAddressesEqual, areApiAddressesEqual } from '~/utils/address'

const t = useNuxtApp().$i18n.t
const rtc = useRuntimeConfig().public
const preexistingCompanyProvisions = rtc.preexistingCompanyProvisions

useHead({
  title: t('transitionApplication.title')
})

const route = useRoute()

definePageMeta({
  layout: 'form',
  middleware: () => {
    // redirect to reg home with return url if user unauthenticated
    const { $keycloak, $config } = useNuxtApp()
    if (!$keycloak.authenticated) {
      const returnUrl = encodeURIComponent(window.location.href)
      return navigateTo(
        `${$config.public.registryHomeUrl}login?return=${returnUrl}`,
        { external: true }
      )
    }
  }
})

const ConnectAddressDisplay = resolveComponent('ConnectAddressDisplay')

const businessId = route.params.businessId as string

const filingStore = usePostRestorationTransitionApplicationStore()
filingStore.init(businessId)
const {
  certifiedByLegalName,
  compPartyEmail,
  courtOrderNumber,
  directors,
  folio,
  legalName,
  offices,
  planOfArrangement,
  regOfficeEmail
} = storeToRefs(filingStore)

const feeStore = useConnectFeeStore()
feeStore.feeOptions.showServiceFees = true
feeStore.fees = {
  OFFICER_CHANGE: {
    filingFees: 10,
    filingType: 'Officer change fee',
    filingTypeCode: 'OFFICER_CHANGE',
    futureEffectiveFees: 0,
    priorityFees: 0,
    processingFees: 0,
    serviceFees: 0,
    tax: {
      gst: 0,
      pst: 0
    },
    total: 0,
    waived: true
  }
}

const sectionErrors = ref({
  reviewAndConfirm: false,
  certify: false
})

// Define table columns
const officesColumns = ref([
  {
    accessorKey: 'officeType',
    header: t('label.office'),
    cell: ({ row }) => {
      return h(
        'div',
        { class: 'text-left font-bold text-bgGovColor-midGray' },
        t(`label.${row.original.officeType}`)
      )
    }
  },
  {
    accessorKey: 'mailingAddress',
    header: t('label.mailingAddress'),
    cell: ({ row }) => { return h(ConnectAddressDisplay, { address: row.original.mailingAddress }) }
  },
  {
    accessorKey: 'deliveryAddress',
    header: t('label.deliveryAddress'),
    cell: ({ row }) => {
      if (areUiAddressesEqual(row.original.deliveryAddress, row.original.mailingAddress)) {
        return t('label.sameAsMailingAddress')
      }
      return h(ConnectAddressDisplay, { address: row.original.deliveryAddress })
    }
  }
])
const directorsColumns = ref([
  {
    accessorKey: 'officer',
    header: t('label.name'),
    cell: ({ row }) => {
      return h(
        'div',
        { class: 'text-left font-bold text-bgGovColor-midGray' },
        `${row.original.officer.firstName} ${row.original.officer.lastName}`
      )
    }
  },
  {
    accessorKey: 'mailingAddress',
    header: t('label.mailingAddress'),
    cell: ({ row }) => {
      return h(ConnectAddressDisplay, { address: formatAddressUi(row.original.mailingAddress) })
    }
  },
  {
    accessorKey: 'deliveryAddress',
    header: t('label.deliveryAddress'),
    cell: ({ row }) => {
      if (areApiAddressesEqual(row.original.deliveryAddress, row.original.mailingAddress)) {
        return t('label.sameAsMailingAddress')
      }
      return h(ConnectAddressDisplay, { address: formatAddressUi(row.original.deliveryAddress) })
    }
  },
  {
    accessorKey: 'startDate',
    header: t('label.effectiveDates'),
    cell: ({ row }) => {
      const directorRole = row.original.roles.find((role: Role) => role.roleType === 'Director')
      const fromDate = fromIsoToUsDateFormat(directorRole.appointmentDate)
      const toDate = fromIsoToUsDateFormat(directorRole.cessationDate) || t('label.current')
      return h(
        'div',
        { class: 'text-left text-wrap' },
        `${fromDate} ${t('label.to')} ${toDate}`
      )
    }
  },
  {
    accessorKey: 'action',
    header: '',
    cell: () => {
      return h(
        'div',
        { class: 'text-left font-bold text-bgGovColor-midGray' },
        'TODO: ADD CHANGE LINK HERE'
      )
    }
  }
])
</script>

<template>
  <div class="py-10 space-y-10">
    <div>
      <h1 class="mb-2">
        {{ $t('transitionApplication.title') }}
      </h1>
      <p class="text-2xl">
        {{ $t('text.transitionYourBusiness') }}
      </p>
    </div>
    <FormSection
      :title="$t('transitionApplication.subtitle.reviewAndConfirm')"
      :description="$t('text.reviewAndConfirmDescription')"
      :has-errors="sectionErrors.reviewAndConfirm"
      class="space-y-4"
    >
      <div class="space-y-6">
        <FormSubSection
          icon="i-mdi-domain"
          :title="$t('label.officeAddresses')"
        >
          <FormDataList
            :data="offices"
            :columns="officesColumns"
            class="m-6"
          />
          <InfoBox
            icon="i-mdi-information-outline"
            :title="$t('text.needChange')"
            :text="$t('text.goToMainFileAddressChange')"
            class="m-6"
          />
        </FormSubSection>

        <FormSubSection
          icon="i-mdi-account-multiple-plus"
          :title="$t('label.currentDirectors')"
        >
          <FormDataList
            :data="directors"
            :columns="directorsColumns"
            class="m-6"
          />
          <InfoBox
            icon="i-mdi-information-outline"
            :title="$t('text.needOtherChange')"
            :text="$t('text.deleteAndFileDirectorChange')"
            class="m-6"
          />
        </FormSubSection>
      </div>
    </FormSection>
    <FormSection
      :title="$t('transitionApplication.subtitle.documentDelivery')"
      :description="$t('text.documentDelivery')"
      :has-errors="sectionErrors.reviewAndConfirm"
      class="space-y-4"
    >
      <FormSubSection
        title=""
        class="space-y-6 p-6"
      >
        <ConnectFormSection :title="$t('label.registeredOffice')">
          {{ regOfficeEmail }}
        </ConnectFormSection>
        <ConnectFormSection :title="$t('label.completingParty')">
          <ConnectFormInput
            v-model="compPartyEmail"
            :name="'documentDelivery.completingPartyEmail'"
            :label="$t('label.emailAddressOptional')"
            :help="$t('text.completingPartyEmail')"
          />
        </ConnectFormSection>
      </FormSubSection>
    </FormSection>
    <FormSection
      :title="$t('transitionApplication.subtitle.courtOrder')"
      :description="$t('text.courtOrder')"
      :has-errors="sectionErrors.reviewAndConfirm"
      class="space-y-4"
    >
      <FormSubSection
        title=""
        class="space-y-6 p-6"
      >
        <ConnectFormSection :title="$t('label.courtOrderNumber')">
          <ConnectFormInput
            v-model="courtOrderNumber"
            :name="'courtOrder.number'"
            :label="$t('label.courtOrderNumber')"
          />
        </ConnectFormSection>
        <ConnectFormSection :title="$t('label.planOfArrangement')">
          <UCheckbox
            v-model="planOfArrangement"
            :label="$t('label.planOfArrangement')"
            :ui="{ base: 'cursor-pointer mt-1', label: 'cursor-pointer', wrapper: 'w-fit' }"
          />
        </ConnectFormSection>
      </FormSubSection>
    </FormSection>
    <FormSection
      :title="$t('transitionApplication.subtitle.folio')"
      :description="$t('text.folioOrReferenceNumber')"
      :has-errors="sectionErrors.reviewAndConfirm"
      class="space-y-4"
    >
      <FormSubSection
        title=""
        class="space-y-6 p-6"
      >
        <ConnectFormSection :title="$t('label.folioOrReferenceNumber')">
          <ConnectFormInput
            v-model="folio"
            :name="'business.folio'"
            :label="$t('label.folioOrReferenceNumberOptional')"
          />
        </ConnectFormSection>
      </FormSubSection>
    </FormSection>
    <FormSection
      :title="$t('transitionApplication.subtitle.companyProvisions')"
      :has-errors="sectionErrors.reviewAndConfirm"
    >
      <FormSubSection
        title=""
        class="p-6"
      >
        <p class="font-bold">
          {{ $t('text.companyProvisionsHeading') }}
        </p>
        <i18n-t
          keypath="text.companyProvisionsText"
          tag="p"
          class="mb-4"
        >
          <template #businessCorporationsAct>
            <em>{{ $t('label.businessCorporationsAct') }}</em>
          </template>
        </i18n-t>
        <ULink
          :to="preexistingCompanyProvisions"
          external
          target="_blank"
          active-class="text-bcGovColor-activeBlue"
          inactive-class="text-bcGovColor-activeBlue"
        >
          {{ $t('text.companyProvisionsURL') }}
        </ULink>
      </FormSubSection>
    </FormSection>
    <FormSection
      :title="$t('transitionApplication.subtitle.certify')"
      :description="$t('text.certifySectionDescription')"
      :has-errors="sectionErrors.certify"
      class="space-y-4"
    >
      <FormSubSection
        title=""
        class="space-y-6 p-6"
      >
        <ConnectFormSection
          :title="$t('label.legalName')"
        >
          <ConnectFormInput
            v-model="legalName"
            :name="'documentDelivery.completingPartyEmail'"
            :label="$t('label.legalName')"
            :placeholder="$t('text.legalNameOfAuthorizedPerson')"
          />
        </ConnectFormSection>
        <ConnectFormSection title=" ">
          <FormCertify
            v-model="certifiedByLegalName"
            :legal-name="legalName"
          />
        </ConnectFormSection>
      </FormSubSection>
    </FormSection>
    <!-- NB: needed so that the tw classes are loaded -->
    <!-- <main class="app-inner-container app-body">
      <div class="flex flex-col lg:flex-row lg:gap-6 grow">
        <div class="grow max-w-full overflow-hidden">
          <slot />
        </div>

        <div class="shrink-0 lg:w-[300px] lg:static sticky lg:mt-10">
          <ConnectFeeWidget class="sticky lg:top-10" />
        </div>
      </div>
    </main> -->
  </div>
</template>
