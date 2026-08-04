<template>
  <div class="flex w-full flex-col overflow-y-auto px-5 pb-5">
    <div
      v-if="codes"
      class="mb-1 mt-4 text-base text-ink-gray-7"
    >
      {{ __('Constituent Codes') }}: <b>{{ codes }}</b>
    </div>
    <div class="overflow-x-auto" :class="{ 'mt-4': !codes }">
      <table class="w-full text-base">
        <thead>
          <tr class="border-b text-left text-sm text-ink-gray-5">
            <th class="py-2 pr-4 font-normal">{{ __('Relationship') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Person / Organization') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Role') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('Reciprocal') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('From') }}</th>
            <th class="py-2 pr-4 font-normal">{{ __('To') }}</th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="(row, i) in rows"
            :key="i"
            class="border-b border-outline-gray-1"
          >
            <td class="whitespace-nowrap py-2 pr-4">{{ row.relationship_type }}</td>
            <td class="max-w-64 truncate py-2 pr-4">
              <!-- related people who exist in the CRM are one click away; name-only
                   relatives (never migrated as donors) render as plain text -->
              <RouterLink
                v-if="row.crm_contact"
                class="underline decoration-outline-gray-3 underline-offset-2 hover:text-ink-gray-9"
                :to="{ name: 'Contact', params: { contactId: row.crm_contact } }"
              >
                {{ row.person }}
              </RouterLink>
              <span v-else>{{ row.person }}</span>
            </td>
            <td class="max-w-48 truncate py-2 pr-4">{{ row.position || '—' }}</td>
            <td class="whitespace-nowrap py-2 pr-4">{{ row.reciprocal_type || '—' }}</td>
            <td class="whitespace-nowrap py-2 pr-4">{{ fdate(row.start_date) }}</td>
            <td class="whitespace-nowrap py-2 pr-4">{{ fdate(row.end_date) }}</td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup>
// One-view (office ask 2026-08-04): the constituent's complete relationship list from
// ERPNext, on the CRM record page. The side panel keeps its clipped one-line summary;
// this tab is the full set (fundraising.crm.giving.donor_history `relationships`).
import { RouterLink } from 'vue-router'
import { formatDate } from '@/utils'

defineProps({
  rows: { type: Array, default: () => [] },
  codes: { type: String, default: '' },
})

// site date_format, same as DonationsListView
function fdate(v) {
  return v ? formatDate(v, '', true) : '—'
}
</script>
