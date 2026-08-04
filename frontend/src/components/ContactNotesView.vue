<template>
  <div class="flex w-full flex-col overflow-y-auto px-5 pb-5">
    <div class="mb-3 mt-4 flex items-center justify-between gap-4">
      <div class="text-sm text-ink-gray-6">
        {{ __('Notes added here are linked to this constituent and copied to their ERPNext record automatically.') }}
      </div>
      <Button
        variant="solid"
        :label="__('New Note')"
        iconLeft="plus"
        @click="openNew"
      />
    </div>
    <div
      v-if="notes.data?.length"
      class="grid grid-cols-1 gap-4 lg:grid-cols-2 xl:grid-cols-3"
    >
      <div
        v-for="note in notes.data"
        :key="note.name"
        class="group flex h-48 cursor-pointer flex-col justify-between gap-2 rounded-md bg-surface-gray-1 px-4 py-3 hover:bg-surface-gray-2"
        @click="openEdit(note)"
      >
        <div class="flex items-center justify-between">
          <div class="truncate text-lg-medium text-ink-gray-8">
            {{ note.title || __('Untitled') }}
          </div>
          <Dropdown
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
        </div>
        <TextEditor
          v-if="note.content"
          :content="note.content"
          :editable="false"
          editor-class="prose-sm text-p-sm max-w-none text-ink-gray-5 focus:outline-none"
          class="flex-1 overflow-hidden"
        />
        <div class="mt-1 flex items-center justify-between gap-2">
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
// carry reference_doctype/docname = this Contact, so the engagement bridge
// (fundraising.crm.engagement) mirrors them to the constituent's ERP Donor Note within
// seconds. The CRM sidebar "Notes" page creates UNLINKED notes (title+content only) —
// this tab is the linked path, on the constituent record where the office looks for it.
import UserAvatar from '@/components/UserAvatar.vue'
import TimelineTimestamp from '@/components/Activities/TimelineTimestamp.vue'
import { usersStore } from '@/stores/users'
import { Button, Dialog, Dropdown, TextEditor, TextInput, call, toast } from 'frappe-ui'
import { ref } from 'vue'

const props = defineProps({
  contactId: { type: String, required: true },
  notes: { type: Object, required: true },
})

const { getUser } = usersStore()

const showDialog = ref(false)
const editingName = ref(null)
const noteTitle = ref('')
const noteContent = ref('')
const saving = ref(false)

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
          reference_doctype: 'Contact',
          reference_docname: props.contactId,
        },
      })
    }
    showDialog.value = false
    props.notes.reload()
  } catch (e) {
    toast.error(e.messages?.[0] || __('Could not save the note'))
  } finally {
    saving.value = false
  }
}

async function deleteNote(name) {
  try {
    await call('frappe.client.delete', { doctype: 'FCRM Note', name })
    props.notes.reload()
  } catch (e) {
    toast.error(e.messages?.[0] || __('Could not delete the note'))
  }
}
</script>
