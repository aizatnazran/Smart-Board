<script setup>
import { supabase } from '@/lib/supaBaseClient'
import Swal from 'sweetalert2'
import { onMounted, ref } from 'vue'

const studentslist = ref([])
const newstudentNumber = ref('')

const fetchstudent = async () => {
  try {
    const teacher_id = localStorage.getItem('teacher_id')
    console.log('Teacher ID: ', teacher_id)
    if (teacher_id) {
      const { data: student, error } = await supabase.from('students').select('*').eq('teacher_id', teacher_id)
      console.log('Fetched students: ', student)
      if (error) {
        console.error('Error fetching students: ', error)
      } else if (student) {
        studentslist.value = student
      }
    }
  } catch (error) {
    console.error(error)
  }
}

const addNewstudent = async () => {
  if (newstudentNumber.value) {
    try {
      const teacher_id = localStorage.getItem('teacher_id')
      if (teacher_id) {
        const requestData = { student_number: newstudentNumber.value, teacher_id }

        const { data, error } = await supabase.from('students').insert([requestData])

        if (error) {
          throw error
        }

        // Ensure that the newly added student, along with its generated id, is pushed to studentslist
        if (data && data.length > 0) {
          console.log('Newly added student data:', data[0]) // Log the newly added student data
          studentslist.value.push(data[0])
        }

        newstudentNumber.value = ''

        Swal.fire({
          title: 'Success!',
          text: 'student successfully added!',
          icon: 'success',
          confirmButtonColor: '#3085d6',
        })
      }
    } catch (error) {
      console.error('Error adding student:', error.message)
      Swal.fire({
        title: 'Error!',
        text: 'Error adding student: ' + error.message,
        icon: 'error',
        confirmButtonColor: '#d33',
      })
    }
  } else {
    Swal.fire({
      title: 'Attention!',
      text: 'Please enter a student number before submitting.',
      icon: 'warning',
      confirmButtonColor: '#3085d6',
    })
  }
  fetchstudent()
}

const editstudent = async student => {
  const { value: newstudentNumber } = await Swal.fire({
    title: 'Enter the new student number',
    input: 'text',
    inputLabel: 'student Number',
    inputValue: student.student_number,
    showCancelButton: true,
    inputValidator: value => {
      if (!value) {
        return 'student number cannot be empty!'
      }
    },
  })

  if (newstudentNumber) {
    try {
      const updatedData = { student_number: newstudentNumber, teacher_id: student.teacher_id, id: student.id }

      const { error } = await supabase.from('students').upsert([updatedData], { onConflict: 'id' })

      if (error) {
        throw error
      }

      const index = studentslist.value.findIndex(c => c.id === student.id)
      if (index !== -1) {
        studentslist.value[index] = updatedData
      }

      Swal.fire('Updated!', 'student updated successfully.', 'success')
    } catch (error) {
      console.error('Error updating student:', error.message)
      Swal.fire('Error!', 'Error updating student: ' + error.message, 'error')
    }
  }
}

// Function to confirm and delete a student
const confirmDelete = async student => {
  Swal.fire({
    title: 'Are you sure?',
    text: 'You will not be able to recover this student!',
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#3085d6',
    cancelButtonColor: '#d33',
    confirmButtonText: 'Yes, delete it!',
  }).then(async result => {
    if (result.isConfirmed) {
      try {
        const { error } = await supabase.from('students').delete().match({ id: student.id })

        if (error) {
          throw error
        }

        studentslist.value = studentslist.value.filter(c => c.id !== student.id)

        Swal.fire('Deleted!', 'student has been deleted.', 'success')
      } catch (error) {
        console.error('Error deleting student:', error.message)
        Swal.fire('Error!', 'Error deleting student: ' + error.message, 'error')
      }
    }
  })
}

onMounted(() => {
  const teacher_id = localStorage.getItem('teacher_id')
  if (teacher_id) {
    fetchstudent()
  }
})
</script>

<template>
  <VCard title="List of Students">
    <VTable class="text-no-wrap">
      <thead>
        <tr>
          <th
            scope="col"
            class="text-start"
          >
            Student Name
          </th>
          <th
            scope="col"
            class="text-start"
          >
            Class
          </th>
          <th
            scope="col"
            class="text-start"
          >
            Age
          </th>
          <th
            scope="col"
            class="text-center"
          >
            Action
          </th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="student in studentslist"
          :key="student.id"
        >
          <td>{{ student.name }}</td>
          <td>{{ student.class }}</td>
          <td>{{ student.age }}</td>
          <td class="text-center">
            <div class="icon-wrapper">
              <VIcon @click="editstudent(student)">mdi-pencil</VIcon>
              <VIcon @click="confirmDelete(student)">mdi-delete</VIcon>
            </div>
          </td>
        </tr>
      </tbody>
    </VTable>
    <VDivider />

    <VCardText>
      <VForm @submit.prevent="() => {}">
        <p class="text-base font-weight-medium mt-5">Add new student</p>

        <VRow>
          <VCol
            cols="12"
            sm="6"
          >
            <VTextField
              label="Student Name"
              v-model="newstudentNumber"
              variant="solo-filled"
            />
          </VCol>
        </VRow>

        <div class="d-flex flex-wrap gap-4 mt-4">
          <VBtn @click="addNewstudent"> Add student </VBtn>
          <VBtn
            color="secondary"
            variant="tonal"
            type="reset"
          >
            Reset
          </VBtn>
        </div>
      </VForm>
    </VCardText>
  </VCard>
</template>

<style scoped>
.text-center {
  text-align: center;
}
.icon-wrapper {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 10px; /* Adjust the gap between icons as needed */
}
</style>
