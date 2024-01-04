<script setup>
import blankImage from '@images/images/blank.png'
import csvImage from '@images/images/csv.png'
import pdfImage from '@images/images/pdf.png'
import xlsImage from '@images/images/xls.png'

import Swal from 'sweetalert2'
import { useTheme } from 'vuetify'
import { supabase } from '../lib/supaBaseClient.js'

// Components

const vuetifyTheme = useTheme()

const selectedFile = ref(null)

const handleFileChange = event => {
  selectedFile.value = event.target.files[0]
  console.log('Selected file:', selectedFile.value)
}

function getImageSrc(fileName) {
  const extension = fileName.split('.').pop()
  switch (extension.toLowerCase()) {
    case 'csv':
      return csvImage
    case 'pdf':
      return pdfImage
    case 'xlsx':
      return xlsImage
    default:
      return blankImage
  }
}

const sheet = ref(false)

// const props = defineProps({

//   data: {
//     sheet: false,
//   },

// })
const userUUID = localStorage.getItem('uuid')
const teacherId = localStorage.getItem('teacher_id')

const uploadFile = async () => {
  if (!selectedFile.value) {
    Swal.fire({
      title: 'Error!',
      text: 'No file selected.',
      icon: 'error',
      customClass: { container: 'high-z-index-swal' },
    })
    return
  }

  // Get the original file name
  const originalFileName = selectedFile.value.name
  console.log('Original file name:', originalFileName)

  // Construct the new file name with uuid and original file name
  const newFileName = `${userUUID}_${originalFileName}`
  console.log('New file name for storage:', newFileName)

  const filePath = `${newFileName}` // Use new file name for the path

  // Upload to Supabase Storage with the new file name
  const { error: uploadError } = await supabase.storage.from('assignments').upload(filePath, selectedFile.value)

  if (uploadError) {
    console.error('Error uploading file:', uploadError)
    Swal.fire({
      title: 'Error!',
      text: 'Error uploading file.',
      icon: 'error',
      customClass: { container: 'high-z-index-swal' },
    })
    return
  }

  // Add a record to your database with the original file name
  const { error: dbError } = await supabase.from('uploadfile').insert([
    {
      uploadfile_filename: originalFileName, // Keep the original file name in the database
      uploadfile_teacher: teacherId,
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
    return
  }

  // Clear the selected file
  selectedFile.value = null

  // Update the files list
  filesList.value = await fetchFiles()

  Swal.fire({
    title: 'Success!',
    text: 'Your file has been uploaded.',
    icon: 'success',
    customClass: { container: 'high-z-index-swal' },
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
      Upload Document
    </VBtn>
  </VRow>
  <div>
    <VContainer>
      <!-- Submitted Document Docs -->
      <VRow>
        <VChip class="mb-3 mt-6">
          <p class="text-title ma-5">Uploaded Assignments</p>
        </VChip>
      </VRow>

      <VRow>
        <VCard
          class="document-card ma-1"
          v-for="file in filesList"
          :key="file.uploadfile_filename"
          cols="6"
        >
          <v-btn
            icon
            flat
            color="transparent"
            class="dots-button"
            v-on="on"
            @click="confirmDelete(file)"
          >
            <v-icon>mdi-dots-vertical</v-icon>
          </v-btn>
          <img
            :src="getImageSrc(file.uploadfile_filename)"
            alt="Document Icon"
          />
          <p class="file-name">{{ file.uploadfile_filename }}</p>
        </VCard>
      </VRow>
    </VContainer>

    <div class="text-center">
      <VDialog
        v-model="sheet"
        activator="parent"
        width="60%"
      >
        <VCard class="pa-10">
          <VContainer
            class="pa-5 rounded-lg mt-2"
            style="background-color: rgb(222, 222, 222)"
          >
            <!--
                <div class="text-overline text-start ml-10">
                File Menu :
                </div> 
              -->
            <div class="text-overline text-start ml-10 text-white">File Upload :</div>
            <VFileInput
              @change="handleFileChange"
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
.document-card {
  width: 180px;
  height: 120px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  opacity: 0.8;
  position: relative;
}

.document-card img {
  width: 50px;
  margin: auto;
}

.file-name {
  text-align: center;
  margin-bottom: 2px;
  padding: 0 10px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 170px;
  font-size: 12px;
  font-weight: bold;
}

.dots-button {
  position: absolute;
  top: 1px;
  right: 1px;
  z-index: 10;
}

.high-z-index-swal {
  z-index: 9999999 !important;
}
</style>
