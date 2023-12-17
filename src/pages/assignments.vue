<script setup>
import csvimg from '@images/images/assignments.png'
import Swal from 'sweetalert2'
import { useTheme } from 'vuetify'
import { supabase } from '../lib/supaBaseClient.js'

// Components

const vuetifyTheme = useTheme()

const selectedFiles = ref([])

const dialog = ref(false)

const handleFileChange = event => {
  selectedFiles.value = Array.from(event.target.files) // Store all selected files
  console.log('Selected files:', selectedFiles.value)
}

const selectedTemplate = ref('template1')

const sheet = ref(false)

// const props = defineProps({

//   data: {
//     sheet: false,
//   },

// })
const userUUID = localStorage.getItem('uuid')
const teacherId = localStorage.getItem('teacher_id')

const currentFile = ref(null)

const handleCardClick = file => {
  currentFile.value = file
  dialog.value = true
}

const uploadFiles = async () => {
  if (!selectedFiles.value.length) {
    await Swal.fire({
      title: 'Error!',
      text: 'No file selected.',
      icon: 'error',
      customClass: { container: 'high-z-index-swal' },
    })
    return
  }

  let uploadErrors = []

  for (const file of selectedFiles.value) {
    const originalFileName = file.name
    const newFileName = `${userUUID}_${originalFileName}`
    const filePath = `${newFileName}`

    // Upload each file to Supabase Storage
    const { error: uploadError } = await supabase.storage.from('assignments').upload(filePath, file)

    if (uploadError) {
      console.error('Error uploading file:', uploadError)
      uploadErrors.push(originalFileName) // Add the file name to the error list
      continue
    }

    // Add record to the database for each file
    const { error: dbError } = await supabase.from('uploadfile').insert([
      {
        uploadfile_filename: originalFileName,
        uploadfile_teacher: teacherId,
        uploadfile_uuid: userUUID,
      },
    ])

    if (dbError) {
      console.error('Error saving file info to database:', dbError)
      uploadErrors.push(originalFileName) // Add the file name to the error list
    }
  }

  // Clear the selected files
  selectedFiles.value = []

  // Update the files list
  filesList.value = await fetchFiles()

  if (uploadErrors.length === 0) {
    Swal.fire({
      title: 'Success!',
      text: 'All your files have been uploaded.',
      icon: 'success',
      customClass: { container: 'high-z-index-swal' },
    })
  } else {
    Swal.fire({
      title: 'Some or all files failed to upload',
      text: `The following files could not be uploaded: ${uploadErrors.join(', ')}`,
      icon: 'warning',
      customClass: { container: 'high-z-index-swal' },
    })
  }
}

async function fetchFiles() {
  let { data: files, error } = await supabase.from('uploadfile').select('*').eq('uploadfile_teacher', teacherId)

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
  filesList.value = await fetchFiles()
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
      Upload Assignment
    </VBtn>
  </VRow>
  <div>
    <VContainer>
      <!-- Submitted Document Docs -->
      <VRow>
        <VChip class="mb-3 mt-6">
          <p class="text-title ma-5">Assignments Uploaded</p>
        </VChip>
      </VRow>

      <VRow>
        <div
          v-if="filesList.length === 0"
          class="text-center my-5"
        >
          <p>No assignments uploaded yet.</p>
        </div>
        <template v-else>
          <template
            v-for="file in filesList"
            :key="file.uploadfile_filename"
          >
            <VCard
              class="document-card ma-1"
              @click="handleCardClick(file)"
            >
              <VBtn
                icon
                class="m-2"
                style="
                  position: absolute;
                  top: 0;
                  right: 0;
                  z-index: 2;
                  width: 24px;
                  height: 24px;
                  min-width: 24px;
                  padding: 0;
                  margin: 2px;
                "
                @click="confirmDelete(file)"
              >
                <VIcon size="x-small">mdi-close</VIcon>
              </VBtn>
              <VImg
                class="ma-1 large-image"
                max-width="100"
                :src="csvimg"
              />
              <p class="text-center text-caption px-2 file-name">{{ file.uploadfile_filename }}</p>
            </VCard>
          </template>
        </template>
        <VDialog v-model="dialog">
          <VCard>
            <VCardTitle>File Details</VCardTitle>
            <VCardText>
              <div>Name: {{ currentFile.value?.uploadfile_filename }}</div>
              <div>Size: {{ currentFile.value?.size }} bytes</div>
              <div>Uploaded: {{ currentFile.value?.created_at }}</div>
            </VCardText>
            <VCardActions>
              <VBtn
                icon
                @click="confirmDelete(currentFile.value)"
              >
                <VIcon>mdi-delete</VIcon>
              </VBtn>
            </VCardActions>
          </VCard>
        </VDialog>
      </VRow>
    </VContainer>

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
              counter
              truncate-length="15"
              multiple
            />
          </VContainer>
          <VBtn
            class="mt-5"
            prepend-icon="mdi-send-variant"
            @click="uploadFiles"
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

.large-image {
  width: 100px; /* Adjust width as needed */
  height: 100px; /* Adjust height as needed */
}

.document-card {
  width: 100px;
  height: 150px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  border-radius: 5%;
  opacity: 0.8;
}

.file-name {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}
</style>
