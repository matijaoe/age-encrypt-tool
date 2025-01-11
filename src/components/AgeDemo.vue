<script lang="ts" setup>
import * as age from 'age-encryption'

const input = ref('')
const ciphertext = ref<Uint8Array>()
const passphrase = ref('test123')
const downloadUrl = ref('')

const getFileName = () => {
  const date = new Date().toISOString().replace(/[:.]/g, '-')
  return `encrypted-${date}.age`
}

const encrypt = async () => {
  try {
    const encrypter = new age.Encrypter()
    encrypter.setPassphrase(passphrase.value)
    ciphertext.value = await encrypter.encrypt(input.value)

    const blob = new Blob([ciphertext.value], {
      type: 'application/octet-stream',
    })

    if (downloadUrl.value) {
      URL.revokeObjectURL(downloadUrl.value)
    }
    downloadUrl.value = URL.createObjectURL(blob)
  } catch (error) {
    console.error('Encryption failed:', error)
  }
}

// Cleanup URL when component is unmounted
onUnmounted(() => {
  if (downloadUrl.value) {
    URL.revokeObjectURL(downloadUrl.value)
  }
})
</script>

<template>
  <div>
    <h1 class="text-primary">age encrypt tool</h1>
    <div class="flex flex-col items-end gap-2 mt-4">
      <UTextarea v-model="input" class="w-full" placeholder="Enter text to encrypt" />
      <div class="flex gap-2">
        <UButton @click="encrypt" :disabled="!input">Encrypt</UButton>

        <UButton
          v-if="downloadUrl"
          :href="downloadUrl"
          :download="getFileName()"
          tag="a"
          target="_blank"
        >
          Download .age file
        </UButton>
      </div>
    </div>
  </div>
</template>
