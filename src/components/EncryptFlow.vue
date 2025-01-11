<script lang="ts" setup>
import * as age from 'age-encryption'
import { breakpointsTailwind, useBreakpoints } from '@vueuse/core'

const input = ref('')

// TODO: can i somehow not have to store passphrase in state?
const passphrase = ref('')
// TODO: signal to user if passphrases was trimmed or modified in some way
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

const isPassphraseValid = computed(() => trimmedPassphrase.value.length > 0)

whenever(metaEnter, () => {
  if (activeStep.value === 0 && input.value) {
    nextStep()
  } else if (activeStep.value === 1 && trimmedPassphrase.value && !passphraseMismatch.value) {
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
    icon: 'i-lucide-file-text',
  },
  {
    title: 'Set passphrase',
    icon: 'i-lucide-key',
  },
  {
    title: 'Encrypt',
    icon: 'i-lucide-lock',
  },
]

const getFileName = () => {
  const timestamp = Date.now()
  return `${timestamp}.age`
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
watch(trimmedPassphrase, (val) => {
  if (!val) {
    passphraseConfirm.value = ''
  }
})

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

const isPassphraseConfirmed = computed(() => {
  return trimmedPassphrase.value && trimmedPassphraseConfirm.value && !passphraseMismatch.value
})

const downloadPassphrase = () => {
  const blob = new Blob([trimmedPassphrase.value], { type: 'text/plain' })
  const timestamp = Date.now()
  downloadBlob(blob, `passphrase_${timestamp}.txt`)
}

const breakpoints = useBreakpoints(breakpointsTailwind)

watch(activeStep, (newStep) => {
  if (newStep === 2) {
    showPassphrase.value = false
    showPassphraseConfirm.value = false
  }
})
</script>

<template>
  <div>
    <h1 class="text-primary text-right">age encrypt tool</h1>

    <UStepper
      :orientation="breakpoints.smallerOrEqual('md') ? 'vertical' : 'horizontal'"
      size="sm"
      v-model="activeStep"
      :items="steps"
      disabled
      class="mt-8 pt-8"
      :ui="{
        root: 'w-full',
        item: breakpoints.smallerOrEqual('md') ? 'group text-start relative w-40' : '',
      }"
    >
      <template #content>
        <div class="ml-4">
          <!-- Step 1: Text Input -->

          <div v-if="activeStep === 0">
            <UTextarea
              aria-label="Text to encrypt"
              autofocus
              v-model="input"
              class="w-full"
              :rows="6"
              autocomplete="off"
              spellcheck="false"
              placeholder="Enter text to encrypt"
            />
            <div class="mt-3 flex justify-between items-center gap-2">
              <div class="text-xs text-gray-500 flex items-center gap-1">
                <UKbd size="sm" value="meta" /><UKbd size="sm" value="enter" />
                to continue
              </div>
              <UButton
                size="sm"
                variant="subtle"
                trailing-icon="i-lucide-arrow-right"
                :disabled="!input"
                @click="nextStep()"
              >
                Continue
              </UButton>
            </div>
          </div>

          <!-- Step 2: Passphrase -->
          <div v-if="activeStep === 1" class="space-y-3 -mt-[calc(28px+12px)]">
            <div class="flex justify-between items-center gap-2">
              <UButton
                size="sm"
                variant="ghost"
                color="neutral"
                icon="i-lucide-arrow-left"
                @click="goToStep0()"
              >
                Back
              </UButton>
            </div>

            <UAlert
              size="sm"
              color="neutral"
              variant="subtle"
              title="Set your passphrase"
              description="Your passphrase will be used to encrypt the content. You will be asked to confirm it. Make sure to save it securely."
              icon="i-lucide-key"
            />

            <UFormField>
              <UInput
                autofocus
                aria-label="Passphrase"
                v-model="passphrase"
                :type="showPassphrase ? 'text' : 'password'"
                class="w-full font-mono"
                placeholder="Enter passphrase"
                data-1p-ignore
                size="sm"
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
                    @click="toggleShowPassphrase()"
                  />
                </template>
              </UInput>
            </UFormField>

            <UFormField v-if="isPassphraseValid" :error="passphraseMismatch">
              <UInput
                aria-label="Confirm passphrase"
                v-model="passphraseConfirm"
                :type="showPassphraseConfirm ? 'text' : 'password'"
                class="w-full font-mono"
                placeholder="Confirm passphrase"
                autocomplete="off"
                data-1p-ignore
                size="sm"
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
              <UButton
                trailing-icon="i-lucide-arrow-right"
                class="ml-auto"
                size="sm"
                variant="subtle"
                v-if="isPassphraseValid && trimmedPassphraseConfirm.length && !passphraseMismatch"
                :disabled="!isPassphraseConfirmed"
                @click="nextStep()"
              >
                Continue
              </UButton>
            </div>
          </div>

          <!-- Step 3: Review & Download -->
          <div v-if="activeStep === 2" class="space-y-3 -mt-[calc(28px+12px)]">
            <div class="flex justify-between items-center gap-2">
              <UButton
                size="sm"
                variant="ghost"
                color="neutral"
                icon="i-lucide-arrow-left"
                @click="prevStep()"
              >
                Back
              </UButton>

              <UButton size="sm" variant="ghost" icon="i-lucide-refresh-cw" @click="resetForm()">
                Start Over
              </UButton>
            </div>

            <UAlert
              size="sm"
              color="primary"
              variant="subtle"
              title="Ready to encrypt"
              description="Your content is ready to be encrypted. The encrypted file will download
                automatically. Save your passphrase—you’ll need it to decrypt."
              icon="i-lucide-lock"
            />

            <div class="space-y-2">
              <div class="flex items-center justify-between gap-2 w-full">
                <div class="text-xs text-gray-500 flex items-center gap-1">
                  <UKbd size="sm" value="meta" /><UKbd size="sm" value="enter" /> to encrypt
                </div>

                <UButton
                  icon="i-lucide-lock"
                  size="sm"
                  :loading="isEncrypting"
                  @click="handleEncrypt"
                  autofocus
                >
                  Encrypt and download
                </UButton>
              </div>
              <div class="flex items-center justify-end gap-2 w-full">
                <UButton
                  icon="i-lucide-download"
                  size="sm"
                  variant="ghost"
                  @click="downloadPassphrase"
                >
                  Download passphrase
                </UButton>
              </div>
            </div>
          </div>
        </div>
      </template>
    </UStepper>
  </div>
</template>
