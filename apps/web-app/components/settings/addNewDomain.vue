<script setup lang="ts">
  import { z } from 'zod';
  import { computed, ref, storeToRefs, useNuxtApp, useToast } from '#imports';
  import { useEEStore } from '~/stores/eeStore';
  import { useRoute } from '#vue-router';

  const { $trpc } = useNuxtApp();
  const orgShortCode = (useRoute().params.orgShortCode ?? '') as string;

  const buttonLoading = ref(false);
  const buttonLabel = ref('Add Domain');
  const newDomainNameValid = ref<boolean | 'remote' | null>(null);
  const newDomainNameValue = ref('');
  const newDomainNameValidationMessage = ref('');
  const newDomainResponseError = ref('');
  const formValid = computed(() => {
    return newDomainNameValid.value === true;
  });

  const emit = defineEmits(['close']);

  async function createNewDomain() {
    const toast = useToast();
    if (newDomainNameValid.value === false) return;
    toast.add({
      id: 'adding_domain',
      title: 'Adding Domain - Please wait',
      description: `This could take up to 1 minute if it's the first domain you're adding.`,
      icon: 'i-ph-thumbs-up'
    });
    buttonLoading.value = true;
    buttonLabel.value = 'Creating domain...';
    const createNewDomainTrpc =
      $trpc.org.mail.domains.createNewDomain.useMutation();
    await createNewDomainTrpc.mutate({
      domainName: newDomainNameValue.value,
      orgShortCode
    });

    if (createNewDomainTrpc.status.value === 'error') {
      buttonLoading.value = false;
      buttonLabel.value = 'Add Domain';
      newDomainNameValid.value = false;
      toast.remove('adding_domain');
      toast.add({
        id: 'domain_add_fail',
        title: 'Domain Creation Failed',
        description: `${newDomainNameValue.value} domain could not be created.`,
        color: 'red',
        icon: 'i-ph-warning-circle',
        timeout: 5000
      });
      return;
    }
    buttonLoading.value = false;
    buttonLabel.value = 'All done';

    toast.remove('adding_domain');
    toast.add({
      id: 'domain_added',
      title: 'Domain Added',
      description: `${newDomainNameValue.value} has been added successfully.`,
      icon: 'i-ph-thumbs-up',
      timeout: 5000
    });

    emit('close');
  }

  const eeStore = useEEStore();
  const { canAddDomain, canAddDomainPending } = storeToRefs(eeStore);
  eeStore.getCanAddDomain();
</script>

<template>
  <div class="flex h-full w-full flex-col items-start">
    <div
      v-if="canAddDomainPending"
      class="bg-base-3 flex w-full flex-row justify-center gap-4 rounded-xl rounded-tl-2xl p-4">
      <UnUiIcon
        name="i-svg-spinners:3-dots-fade"
        size="24" />
      <span>Checking status</span>
    </div>
    <div
      v-if="!canAddDomainPending && canAddDomain"
      class="flex w-full flex-col gap-4">
      <UnUiInput
        v-model:value="newDomainNameValue"
        v-model:valid="newDomainNameValid"
        v-model:validationMessage="newDomainNameValidationMessage"
        label="Domain name"
        placeholder=""
        :schema="z.string().trim().min(4).includes('.')" />
      <UnUiButton
        :label="buttonLabel"
        :loading="buttonLoading"
        :disabled="!formValid"
        size="sm"
        @click="createNewDomain()" />
      <div
        v-if="newDomainResponseError"
        class="bg-red-3 w-fit rounded-lg px-4 py-1">
        {{ newDomainResponseError }}
      </div>
    </div>
    <div
      v-if="!canAddDomainPending && !canAddDomain"
      class="flex w-full flex-col gap-4">
      <span class="text-xl font-semibold">Add a new domain</span>
      <span>
        Sorry, your current billing plan does not support adding custom domains.
      </span>
    </div>
  </div>
</template>
