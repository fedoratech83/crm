<template>
  <div class="flex w-full flex-col overflow-y-auto px-5 pb-5">
    <div
      v-if="summary && summary.donor"
      class="mb-1 mt-4 flex flex-wrap gap-x-6 gap-y-1 text-base text-ink-gray-7"
    >
      <span>{{ __('Lifetime') }}: <b>{{ fmt(summary.lifetime) }}</b></span>
      <span>{{ __('This FY') }}: <b>{{ fmt(summary.fytd) }}</b></span>
      <span v-if="summary.open_pledge_balance" class="text-amber-700">
        {{ __('Open pledges') }}: <b>{{ fmt(summary.open_pledge_balance) }}</b>
      </span>
      <span>{{ __('Records') }}: <b>{{ summary.total }}</b></span>
      <!-- Office fix 2026-07-23: a spouse/household member whose giving is all soft
           credits (for example Joe Donahue, credited on Lauren's gifts) showed
           "Lifetime $0 / No Donations Found" even though the credits exist in ERPNext.
           Surface the soft-credit position right in the summary strip. -->
      <span v-if="summary.soft_credit_total" class="text-ink-gray-6">
        {{ __('Soft credits') }}: <b>{{ fmt(summary.soft_credit_total) }}</b>
      </span>
    </div>
    <div class="overflow-x-auto">
      <table class="w-full text-base">
        <thead>
          <tr class="border-b text-left text-sm text-ink-gray-5">
            <th class="py-2 pr-4 font-normal">{{ __('Date') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Type') }}</th>
            <th class="py-2 pr-4 text-right font-normal">{{ __('Amount') }}</th>
            <th class="py-2 pr-4 text-right font-normal">{{ __('Open') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Fund') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Status') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Acknowledged') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Receipt') }}</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="row in rows"
            :key="row.name"
            class="cursor-pointer border-b border-outline-gray-1 hover:bg-surface-gray-1"
            :title="row.name"
            @click="openGift(row)"
          >
            <td class="whitespace-nowrap py-2 pr-4">{{ fdate(row.donation_date) }}</td>
            <td class="whitespace-nowrap py-2 pr-4">{{ spaced(row.donation_type) }}</td>
            <td class="whitespace-nowrap py-2 pr-4 text-right">{{ fmt(row.amount) }}</td>
            <td class="whitespace-nowrap py-2 pr-4 text-right">
              <span v-if="isPledge(row) && row.balance" class="text-amber-700">{{ fmt(row.balance) }}</span>
              <span v-else>—</span>
            </td>
            <td class="max-w-48 truncate py-2 pr-4">{{ row.primary_fund || '—' }}</td>
            <td class="whitespace-nowrap py-2 pr-4">
              {{ row.gift_status }}<span v-if="isPledge(row) && row.pledge_stage"> · {{ row.pledge_stage }}</span>
            </td>
            <!-- Office fix 2026-07-23: staff read "Not Receipted" as "never thanked".
                 Acknowledgment (the thank-you letter, migrated from RE: 42k gifts) and
                 the tax receipt document are different things; show the ack status the
                 API already returns, and stop rendering the words "Not Receipted"
                 (RE tracked receipts for under 3% of gifts, so the words were noise). -->
            <td class="whitespace-nowrap py-2 pr-4">{{ spaced(row.acknowledgement_status) || '—' }}</td>
            <td class="whitespace-nowrap py-2 pr-4">
              {{ row.receipt_status === 'NotReceipted' ? '—' : (spaced(row.receipt_status) || '—') }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
    <!-- Soft credits (household / recognition): gifts made by someone else that this
         person is credited on. Read-only context; clicking opens the underlying gift. -->
    <div v-if="summary && summary.soft_rows && summary.soft_rows.length" class="mt-4">
      <div class="mb-1 text-sm font-medium text-ink-gray-5">
        {{ __('Soft credits') }} ({{ summary.soft_rows.length }})
      </div>
      <div class="overflow-x-auto">
        <table class="w-full text-base">
          <thead>
            <tr class="border-b text-left text-sm text-ink-gray-5">
              <th class="py-2 pr-4 font-normal">{{ __('Date') }}</th>
              <th class="py-2 pr-4 font-normal">{{ __('From') }}</th>
              <th class="py-2 pr-4 font-normal">{{ __('Type') }}</th>
              <th class="py-2 pr-4 text-right font-normal">{{ __('Amount') }}</th>
              <th class="py-2 pr-4 font-normal">{{ __('Fund') }}</th>
              <th class="py-2 pr-4 font-normal">{{ __('Recognition') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="sr in summary.soft_rows"
              :key="sr.gift + sr.recognition_type"
              class="cursor-pointer border-b border-outline-gray-1 hover:bg-surface-gray-1"
              :title="sr.gift"
              @click="openGift({ name: sr.gift })"
            >
              <td class="whitespace-nowrap py-2 pr-4">{{ fdate(sr.donation_date) }}</td>
              <td class="max-w-48 truncate py-2 pr-4">{{ sr.from_donor_name || sr.from_donor }}</td>
              <td class="whitespace-nowrap py-2 pr-4">{{ spaced(sr.donation_type) }}</td>
              <td class="whitespace-nowrap py-2 pr-4 text-right">{{ fmt(sr.amount) }}</td>
              <td class="max-w-48 truncate py-2 pr-4">{{ sr.primary_fund || '—' }}</td>
              <td class="whitespace-nowrap py-2 pr-4">{{ spaced(sr.recognition_type) || '—' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <Button
      v-if="summary && rows.length < summary.total"
      class="mt-3 self-start"
      :label="__('Load more')"
      @click="$emit('loadMore')"
    />
  </div>
</template>

<script setup>
// Holy Trinity deploy patch: the donor page's main tab shows the FULL donation history
// from ERPNext (fundraising.crm.giving.donor_history) instead of the sales Deals list.
import { Button } from 'frappe-ui'
import { formatDate } from '@/utils'

defineProps({
  rows: { type: Array, default: () => [] },
  summary: { type: Object, default: null },
})
defineEmits(['loadMore'])

const usd = new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' })

function fmt(v) {
  return usd.format(v || 0)
}
// Office fix 2026-07-23: render dates in the site's date_format (mm-dd-yyyy) instead
// of the raw ISO string the API returns — staff read 2026-03-31 as ambiguous.
function fdate(v) {
  return v ? formatDate(v, '', true) : ''
}
function spaced(v) {
  return (v || '').replace(/([a-z])([A-Z])/g, '$1 $2')
}
function isPledge(row) {
  return ['Pledge', 'MatchingGiftPledge'].includes(row.donation_type)
}
function openGift(row) {
  window.open('/app/ht-donation/' + encodeURIComponent(row.name), '_blank')
}
</script>
