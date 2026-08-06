<template>
  <div class="flex w-full flex-col overflow-y-auto px-5 pb-5">
    <div class="mb-3 mt-4 flex items-center justify-between gap-4">
      <div class="text-sm text-ink-gray-6">
        {{ __("New notes are linked to this constituent and copied to their ERPNext record automatically. Notes from Raiser's Edge and the ERP are read-only here; open them in ERPNext to edit.") }}
      </div>
      <Button
        variant="solid"
        :label="__('New Note')"
        iconLeft="plus"
        @click="openNew"
      />
    </div>
    <div
      v-if="allNotes.length"
      class="grid grid-cols-1 gap-4 lg:grid-cols-2 xl:grid-cols-3"
    >
      <div
        v-for="note in allNotes"
        :key="note._key"
        class="group flex h-48 cursor-pointer flex-col justify-between gap-2 rounded-md bg-surface-gray-1 px-4 py-3 hover:bg-surface-gray-2"
        @click="note._editable ? openEdit(note) : openInErp(note)"
      >
        <div class="flex items-center justify-between gap-2">
          <div class="truncate text-lg-medium text-ink-gray-8">
            {{ note._editable ? note.title || __('Untitled') : note.note_type || __('Note') }}
          </div>
          <Dropdown
            v-if="note._editable"
            :options="[
              {
                label: __('Delete'),
                icon: 'trash-2',
                onClick: () => deleteNote(note.name),
              },
            ]"
            class="h-6 w-6"
            @click.stop
          >
            <Button
              icon="lucide-more-horizontal"
              variant="ghosted"
              class="!h-6 !w-6 hover:bg-surface-gray-2"
              @click.stop.prevent
            />
          </Dropdown>
          <Badge v-else :label="note._source" theme="gray" variant="subtle" class="shrink-0" />
        </div>
        <TextEditor
          v-if="note._editable && note.content"
          :content="note.content"
          :editable="false"
          editor-class="prose-sm text-p-sm max-w-none text-ink-gray-5 focus:outline-none"
          class="flex-1 overflow-hidden"
        />
        <div
          v-else-if="!note._editable"
          class="flex-1 overflow-hidden text-p-sm text-ink-gray-5"
        >
          <!-- migrated RE notes commonly carry BOTH: summary is a short label/caption,
               note_text is the real paragraph content. Showing only one hid whichever
               wasn't picked -- e.g. a "Meeting with Henry and Leslie" summary hid the
               actual multi-sentence meeting account underneath it (office ask 08-06). -->
          <div
            v-if="note.summary && note.note_text"
            class="mb-1 truncate font-medium text-ink-gray-7"
            :title="note.summary"
          >
            {{ note.summary }}
          </div>
          <div class="whitespace-pre-line">
            {{ note.note_text || note.summary || '' }}
          </div>
        </div>
        <div class="mt-1 flex items-center justify-between gap-2">
          <template v-if="note._editable">
            <div class="flex items-center gap-2 truncate">
              <UserAvatar :user="note.owner" size="xs" />
              <div
                class="truncate text-sm text-ink-gray-8"
                :title="getUser(note.owner).full_name"
              >
                {{ getUser(note.owner).full_name }}
              </div>
            </div>
            <TimelineTimestamp
              :date="note.modified"
              class-name="truncate text-sm text-ink-gray-7"
            />
          </template>
          <template v-else>
            <div class="truncate text-sm text-ink-gray-6">{{ __('Open in ERPNext') }}</div>
            <div class="truncate text-sm text-ink-gray-7">{{ fdate(note.note_date || note.creation) }}</div>
          </template>
        </div>
      </div>
    </div>
    <div
      v-else
      class="flex flex-1 items-center justify-center text-base text-ink-gray-4"
    >
      {{ __('No Notes Found') }}
    </div>
    <Dialog
      v-model="showDialog"
      :options="{ title: editingName ? __('Edit Note') : __('New Note') }"
    >
      <template #body-content>
        <TextInput
          v-model="noteTitle"
          :placeholder="__('Title')"
          class="mb-3 w-full"
        />
        <TextEditor
          variant="outline"
          :content="noteContent"
          :editable="true"
          :bubbleMenu="true"
          :placeholder="__('Take a note...')"
          editor-class="!prose-sm overflow-auto min-h-[180px] max-h-80 py-1.5 px-2"
          @change="(val) => (noteContent = val)"
        />
      </template>
      <template #actions>
        <Button
          variant="solid"
          :label="__('Save')"
          :loading="saving"
          class="w-full"
          @click="saveNote"
        />
      </template>
    </Dialog>
  </div>
</template>

<script setup>
// Office ask 2026-08-04 ("How do I add a note to a constituent?"): notes created here
// carry reference_doctype/docname = this record, so the engagement bridge
// (fundraising.crm.engagement) mirrors them to the ERP Donor Note within seconds. The
// CRM sidebar "Notes" page creates UNLINKED notes (title+content only) — this tab is
// the linked path, on the constituent record where the office looks for it.
//
// Office ask 2026-08-05 ("were the RE notes ever moved, or do we just not have the
// view?"): they were all moved (verified 1:1 against the Raiser's Edge backup, zero
// lost) — this tab simply never showed them, since it only ever queried FCRM Note
// (the CRM-native doctype). It now merges CRM notes (editable) with the donor's
// ERP-side Donor Notes (migrated-from-RE or entered directly in ERPNext, both
// read-only here — ERPNext stays the system of record for those).
import UserAvatar from '@/components/UserAvatar.vue'
import TimelineTimestamp from '@/components/Activities/TimelineTimestamp.vue'
import { usersStore } from '@/stores/users'
import { formatDate } from '@/utils'
import { Badge, Button, Dialog, Dropdown, TextEditor, TextInput, call, toast } from 'frappe-ui'
import { computed, ref } from 'vue'

const props = defineProps({
  refId: { type: String, required: true },
  refDoctype: { type: String, default: 'Contact' },
  crmNotes: { type: Object, required: true },
  donorNotes: { type: Object, required: true },
})

const { getUser } = usersStore()

const showDialog = ref(false)
const editingName = ref(null)
const noteTitle = ref('')
const noteContent = ref('')
const saving = ref(false)

const allNotes = computed(() => {
  const crm = (props.crmNotes.data || []).map((n) => ({
    ...n,
    _key: 'crm:' + n.name,
    _editable: true,
    _sortDate: n.modified,
  }))
  const donor = (props.donorNotes.data || []).map((n) => ({
    ...n,
    _key: 'donor:' + n.name,
    _editable: false,
    _source: n.source,
    _sortDate: n.note_date || n.creation,
  }))
  return [...crm, ...donor].sort(
    (a, b) => new Date(b._sortDate || 0) - new Date(a._sortDate || 0),
  )
})

// site date_format, same pattern as the other list views
function fdate(v) {
  return v ? formatDate(v, '', true) : ''
}

function openNew() {
  editingName.value = null
  noteTitle.value = ''
  noteContent.value = ''
  showDialog.value = true
}

function openEdit(note) {
  editingName.value = note.name
  noteTitle.value = note.title || ''
  noteContent.value = note.content || ''
  showDialog.value = true
}

function openInErp(note) {
  window.open('/app/donor-note/' + encodeURIComponent(note.name), '_blank')
}

async function saveNote() {
  if (!noteTitle.value && !noteContent.value) return
  saving.value = true
  try {
    if (editingName.value) {
      await call('frappe.client.set_value', {
        doctype: 'FCRM Note',
        name: editingName.value,
        fieldname: { title: noteTitle.value, content: noteContent.value },
      })
    } else {
      await call('frappe.client.insert', {
        doc: {
          doctype: 'FCRM Note',
          title: noteTitle.value,
          content: noteContent.value,
          reference_doctype: props.refDoctype,
          reference_docname: props.refId,
        },
      })
    }
    showDialog.value = false
    props.crmNotes.reload()
  } catch (e) {
    toast.error(e.messages?.[0] || __('Could not save the note'))
  } finally {
    saving.value = false
  }
}

async function deleteNote(name) {
  try {
    await call('frappe.client.delete', { doctype: 'FCRM Note', name })
    props.crmNotes.reload()
  } catch (e) {
    toast.error(e.messages?.[0] || __('Could not delete the note'))
  }
}
</script>
