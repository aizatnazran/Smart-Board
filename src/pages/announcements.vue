<script setup>
import Swal from 'sweetalert2'
import { useTheme } from 'vuetify'
import { supabase } from '../lib/supaBaseClient.js'

const truncate = (text, length, suffix = '...') => {
  if (text.length > length) {
    return text.substring(0, length) + suffix
  }
  return text
}

// Components

const announcementsList = ref([])

const showAnnouncementDialog = ref(false)
const selectedAnnouncement = ref({})
const vuetifyTheme = useTheme()
const sheet = ref(false)

// const props = defineProps({

//   data: {
//     sheet: false,
//   },

// })
const userUUID = localStorage.getItem('uuid')
const teacherId = localStorage.getItem('teacher_id')

async function fetchAnnouncements() {
  let { data: announcements, error } = await supabase
    .from('announcement')
    .select(
      `
      id,
      announcement_desc,
      created_at,
      teacher:announcement_teacher ( name )
    `,
    ) // Assuming you have set up foreign key relation correctly
    .order('created_at', { ascending: false })

  if (error) {
    console.log('Error fetching announcements:', error)
    return []
  }
  // Process data if necessary, e.g., formatting dates
  console.log(announcements)
  return announcements
}

const openAnnouncement = announcement => {
  console.log('Selected announcement:', announcement)

  // Ensure that the announcement has a teacher property before trying to open the dialog
  if (!announcement.teacher) {
    console.error('Announcement does not have teacher information:', announcement)
    return
  }

  selectedAnnouncement.value = announcement
  showAnnouncementDialog.value = true
}

const uploadFile = async () => {
  if (!selectedFile.value) {
    await Swal.fire({
      title: 'Error!',
      text: 'No file selected.',
      icon: 'error',
      customClass: {
        container: 'high-z-index-swal',
      },
    })
    return
  }

  // Upload to Supabase Storage with the new file name
  const { error: uploadError } = await supabase.storage.from('assignments').upload(filePath, selectedFile.value)

  if (uploadError) {
    console.error('Error uploading file:', uploadError)
    Swal.fire({
      title: 'Error!',
      text: 'Error uploading file.',
      icon: 'error',
      customClass: {
        container: 'high-z-index-swal',
      },
    })
    return
  }

  const { error: dbError } = await supabase.from('uploadfile').insert([
    {
      uploadfile_filename: originalFileName, // Keep the original file name in the database
      uploadfile_company: teacher_id,
      uploadfile_uuid: userUUID,
    },
  ])

  if (dbError) {
    console.error('Error saving file info to database:', dbError)
    Swal.fire({
      title: 'Error!',
      text: 'Error saving file info to database.',
      icon: 'error',
      customClass: { container: 'high-z-index-swal' },
    })
  }

  selectedFile.value = null

  // Update the files list
  filesList.value = await fetchFiles()

  Swal.fire({
    title: 'Success!',
    text: 'Your file has been uploaded.',
    icon: 'success',
    customClass: { container: 'high-z-index-swal' },
    customClass: {
      container: 'high-z-index-swal',
    },
  })
}

async function fetchFiles() {
  let { data: files, error } = await supabase
    .from('uploadfile')
    .select('uploadfile_filename')
    .eq('uploadfile_teacher', teacherId)

  if (error) console.log('Error fetching files:', error)
  else return files
}

const filesList = ref([])

const confirmDelete = async file => {
  Swal.fire({
    title: 'Are you sure?',
    text: `You won't be able to revert the deletion of ${file.uploadfile_filename}!`,
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#3085d6',
    cancelButtonColor: '#d33',
    confirmButtonText: 'Yes, delete it!',
  }).then(async result => {
    if (result.isConfirmed) {
      try {
        let { error } = await supabase
          .from('uploadfile')
          .delete()
          .match({ uploadfile_filename: file.uploadfile_filename, uploadfile_teacher: teacherId })

        if (error) throw error

        // Remove the file from the filesList
        filesList.value = filesList.value.filter(f => f.uploadfile_filename !== file.uploadfile_filename)

        Swal.fire({
          title: 'Deleted!',
          text: 'Your file has been deleted.',
          icon: 'success',
        })
      } catch (error) {
        Swal.fire({
          title: 'Error!',
          text: 'There was a problem deleting your file.',
          icon: 'error',
        })
      }
    }
  })
}

onMounted(async () => {
  announcementsList.value = await fetchAnnouncements()
})
</script>

<template>
  <VRow>
    <VSpacer />
    <VBtn
      color="primary"
      v-bind="props"
      @click="sheet = !sheet"
    >
      Create Announcement
    </VBtn>
  </VRow>
  <div>
    <VContainer>
      <!-- Submitted Document Docs -->
      <VRow>
        <VChip class="mb-3 mt-6">
          <p class="text-title ma-5">Announcements</p>
        </VChip>
      </VRow>

      <VRow>
        <VCard
          class="announcement-card ma-1"
          v-for="announcement in announcementsList"
          :key="announcement.id"
          @click="openAnnouncement(announcement)"
          :cols="12"
          :sm="6"
          :md="6"
          :xl="6"
          :lg="6"
        >
          <div class="announcement-content">
            <!-- Use optional chaining to safely access nested properties -->
            <h3>{{ announcement.teacher?.name }}</h3>
            <p>{{ truncate(announcement.announcement_desc, 100) }}</p>
            <p class="timestamp">{{ new Date(announcement.created_at).toLocaleString() }}</p>
          </div>
        </VCard>
      </VRow>
    </VContainer>
    <VDialog
      v-model="showAnnouncementDialog"
      persistent
      max-width="600px"
    >
      <VCard>
        <!-- Use optional chaining to safely access nested properties -->
        <VCardTitle>{{ selectedAnnouncement.value.teacher?.name }}</VCardTitle>
        <VCardText>{{ selectedAnnouncement.value.announcement_desc }}</VCardText>
        <VCardActions>
          <VSpacer />
          <VBtn
            text
            @click="showAnnouncementDialog = false"
            >Close</VBtn
          >
        </VCardActions>
      </VCard>
    </VDialog>
    <div class="text-center">
      <VDialog
        v-model="sheet"
        activator="parent"
        width="60%"
        class="overlaying-component-class"
      >
        <VCard class="pa-10">
          <VContainer
            class="pa-5 rounded-lg mt-2 border-2"
            style="background-color: var(--v-theme-on-surface); border: 2px solid #8a8d93"
          >
            <!--
                <div class="text-overline text-start ml-10">
                File Menu :
                </div> 
              -->
            <div class="text-overline text-start ml-10 text-title">File Upload :</div>
            <VFileInput
              @change="handleFileChange"
              z
              counter
              truncate-length="15"
            />
          </VContainer>
          <VBtn
            class="mt-5"
            prepend-icon="mdi-send-variant"
            @click="uploadFile"
          >
            Upload
          </VBtn>
        </VCard>
      </VDialog>
    </div>
  </div>
</template>

<style>
.high-z-index-swal {
  z-index: 9999999 !important;
}
.overlaying-component-class {
  z-index: 1050;
}

.announcement-card {
  padding: 15px;
  width: 40%;
}

.announcement-content h3 {
  font-weight: bold;
}

.announcement-content p {
  margin: 5px 0;
}

.timestamp {
  font-size: 0.8em;
  color: #777;
}
</style>
