<script lang="ts" setup>
import * as age from 'age-encryption'

const input = ref('')

const passphrase = ref('')
// TODO: signal to user if pasphrases was trimmed or modified in some way
const trimmedPassphrase = computed(() => passphrase.value?.trim() || '')

const passphraseConfirm = ref('')
const trimmedPassphraseConfirm = computed(() => passphraseConfirm.value?.trim() || '')

const [showPassphrase, toggleShowPassphrase] = useToggle(false)
const [showPassphraseConfirm, toggleShowPassphraseConfirm] = useToggle(false)

const isEncrypting = ref(false)

const {
  count: activeStep,
  inc: nextStep,
  dec: prevStep,
  set: stepStep,
  reset: resetStep,
} = useCounter(0, { min: 0, max: 2 })

const { meta_enter: metaEnter, escape: esc } = useMagicKeys()

whenever(metaEnter, () => {
  if (activeStep.value === 0 && input.value) {
    nextStep()
  } else if (
    activeStep.value === 1 &&
    trimmedPassphrase.value &&
    trimmedPassphrase.value === trimmedPassphraseConfirm.value
  ) {
    nextStep()
  } else if (activeStep.value === 2 && !isEncrypting.value) {
    handleEncrypt()
  }
})

whenever(esc, () => {
  if (activeStep.value > 0) {
    prevStep()
  }
})

// TODO: dont allow skipping steps forward
const steps = [
  {
    title: 'Enter text',
    // description: 'Enter or paste the text you want to encrypt',
    icon: 'i-lucide-file-text',
  },
  {
    title: 'Set Passphrase',
    // description: 'Enter a passphrase to encrypt your text',
    icon: 'i-lucide-key',
  },
  {
    title: 'Encrypt',
    // description: 'Review and encrypt your text',
    icon: 'i-lucide-lock',
  },
]

const getFileName = () => {
  const date = new Date().toISOString().replace(/[:.]/g, '-')
  return `encrypted-${date}.age`
}

const downloadBlob = (blob: Blob, filename: string) => {
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = filename
  document.body.appendChild(a)
  a.click()
  document.body.removeChild(a)
  URL.revokeObjectURL(url)
}

const createEncryptedBlob = async (text: string, passphrase: string): Promise<Blob> => {
  const encrypter = new age.Encrypter()
  encrypter.setPassphrase(passphrase)
  const ciphertext = await encrypter.encrypt(text)
  return new Blob([ciphertext], { type: 'application/octet-stream' })
}

const resetForm = () => {
  input.value = ''
  passphrase.value = ''
  passphraseConfirm.value = ''
  resetStep()
}

const handleEncrypt = async () => {
  try {
    isEncrypting.value = true
    const blob = await createEncryptedBlob(input.value, trimmedPassphrase.value)
    downloadBlob(blob, getFileName())
  } catch (error) {
    console.error('Encryption failed:', error)
  } finally {
    isEncrypting.value = false
  }
}

const passphraseMismatch = computed(() => {
  if (!passphraseConfirm.value?.trim()) {
    return false
  }

  return passphrase.value?.trim() !== passphraseConfirm.value?.trim()
})

const handlePassphraseKeydown = async (e: KeyboardEvent) => {
  if (e.key === 'Enter' && passphrase.value?.trim()) {
    // focus on passphrase confirm input
  }
}

const handlePassphraseConfirmKeydown = async (e: KeyboardEvent) => {
  if (e.key === 'Enter' && passphrase.value?.trim() && !passphraseMismatch.value) {
    nextStep()
  }
}

const clearPassphrases = () => {
  passphrase.value = ''
  passphraseConfirm.value = ''
}

const goToStep0 = () => {
  clearPassphrases()
  stepStep(0)
}

const isPassphraseValid = computed(() => trimmedPassphrase.value.length > 0)

const isPassphraseConfirmed = computed(() => {
  return (
    trimmedPassphrase.value &&
    trimmedPassphraseConfirm.value &&
    trimmedPassphrase.value === trimmedPassphraseConfirm.value
  )
})

const downloadPassphrase = () => {
  const blob = new Blob([trimmedPassphrase.value], { type: 'text/plain' })
  const date = new Date().toISOString().replace(/[:.]/g, '-')
  downloadBlob(blob, `passphrase-${date}.txt`)
}
</script>

<template>
  <div>
    <h1 class="text-primary">age encrypt tool</h1>

    <UStepper disabled size="sm" v-model="activeStep" :items="steps" class="mt-8">
      <template #content>
        <div class="mt-4 p-4">
          <!-- Step 1: Text Input -->
          <div v-if="activeStep === 0">
            <UTextarea
              autofocus
              v-model="input"
              class="w-full font-mono"
              :rows="6"
              placeholder="Enter text to encrypt"
            />
            <div class="mt-2 flex justify-between items-center gap-2">
              <div class="text-xs text-gray-500 flex items-center gap-1">
                <UKbd size="sm" value="meta" /><UKbd size="sm" value="enter" />
                to continue
              </div>
              <UButton size="sm" variant="outline" :disabled="!input" @click="nextStep()">
                Continue
              </UButton>
            </div>
          </div>

          <!-- Step 2: Passphrase -->
          <div v-if="activeStep === 1" class="space-y-4">
            <UFormField>
              <UInput
                autofocus
                v-model="passphrase"
                :type="showPassphrase ? 'text' : 'password'"
                class="w-full font-mono"
                placeholder="Enter passphrase"
                data-1p-ignore
                @keydown="handlePassphraseKeydown"
              >
                <template #trailing>
                  <UButton
                    size="sm"
                    tabindex="-1"
                    color="neutral"
                    variant="link"
                    :icon="showPassphrase ? 'i-lucide-eye-off' : 'i-lucide-eye'"
                    :aria-label="showPassphrase ? 'Hide password' : 'Show password'"
                    :aria-pressed="showPassphrase"
                    data-1p-ignore
                    @click="toggleShowPassphrase()"
                  />
                </template>
              </UInput>
            </UFormField>

            <UFormField v-if="isPassphraseValid" :error="passphraseMismatch">
              <UInput
                v-model="passphraseConfirm"
                :type="showPassphraseConfirm ? 'text' : 'password'"
                class="w-full font-mono"
                placeholder="Confirm passphrase"
                autocomplete="off"
                data-1p-ignore
                @keydown="handlePassphraseConfirmKeydown"
              >
                <template #trailing>
                  <UButton
                    size="sm"
                    tabindex="-1"
                    color="neutral"
                    variant="link"
                    :icon="showPassphraseConfirm ? 'i-lucide-eye-off' : 'i-lucide-eye'"
                    :aria-label="showPassphraseConfirm ? 'Hide password' : 'Show password'"
                    :aria-pressed="showPassphraseConfirm"
                    @click="toggleShowPassphraseConfirm()"
                  />
                </template>
              </UInput>
            </UFormField>

            <div class="flex justify-between gap-2">
              <UButton size="sm" variant="ghost" @click="goToStep0"> Back </UButton>
              <UButton
                size="sm"
                variant="outline"
                v-if="isPassphraseValid && trimmedPassphraseConfirm.length && !passphraseMismatch"
                :disabled="!isPassphraseConfirmed"
                @click="nextStep()"
              >
                Confirm
              </UButton>
            </div>
          </div>

          <!-- Step 3: Review & Download -->
          <div v-if="activeStep === 2" class="space-y-4">
            <UAlert
              color="primary"
              variant="subtle"
              title="Ready to encrypt"
              description="Your text will be encrypted and saved with a unique name based on the current timestamp. Make sure to save your passphrase in a secure location - you'll need it to decrypt the file later."
              icon="i-lucide-lock"
            />

            <div class="flex justify-between items-center gap-2">
              <UButton size="sm" variant="ghost" @click="prevStep()"> Back </UButton>

              <div class="flex gap-2">
                <UButton
                  size="sm"
                  variant="ghost"
                  icon="i-lucide-download"
                  @click="downloadPassphrase"
                >
                  Save Passphrase
                </UButton>
                <div class="text-xs text-gray-500 flex items-center gap-1">
                  <UKbd size="sm" value="meta" /><UKbd size="sm" value="enter" /> to encrypt
                </div>
                <UButton variant="outline" size="sm" :loading="isEncrypting" @click="handleEncrypt">
                  Encrypt
                </UButton>
              </div>
            </div>

            <div>
              <UButton size="sm" variant="ghost" @click="resetForm()"> Start Over </UButton>
            </div>
          </div>
        </div>
      </template>
    </UStepper>
  </div>
</template>
