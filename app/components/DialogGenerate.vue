<template>
  <BaseDialog
    :id="id"
    :open="open"
    @close="emit('close')"
  >
    <template #title>Save lineage</template>
    <Feedback
      ref="status"
      :global-settings="{ showDismiss: false }"
    />
    <div
      v-if="isLoadedAndOk"
      class="flex flex-col"
    >
      <p>
        To share this lineage with other people, copy and paste the link below.
      </p>
      <p>
        Please note if this link is not viewed in 2 months, it will be deleted
        from the server.
      </p>
      <Textbox
        v-model="viewLink"
        readonly
        type="input"
        placeholder="link"
        show-copy-button
        show-share-button
        select-all-on-focus
        copy-button-title="Copy lineage link"
        :share-params="{
          buttonTitle: 'Share lineage',
          title: 'View my lineage',
          text: 'View my DragCave lineage on Lineage Builder',
        }"
      />
    </div>
    <div v-else-if="problemDragon">
      <DragonProblem
        :dragon="problemDragon"
        :highlight="field"
        :error="error"
      />
    </div>
    <template #footer>
      <button
        class="dialog-footer-button"
        @click="emit('close')"
      >
        Close
      </button>
    </template>
  </BaseDialog>
</template>
<script setup lang="ts">
import { onUpdated, ref } from 'vue';
import type { PropType } from 'vue';
import { ValidationError } from 'yup';
import type {
  MaybePartialLineageWithMetadata,
  PartialLineage,
} from '../shared/types';
import { Lineage } from '../shared/lineageHandler';
import BaseDialog from './BaseDialog.vue';
import Feedback from './Feedback.vue';
import Textbox from './Textbox.vue';
import DragonProblem from './DragonProblem.vue';
import { FetchError } from 'ofetch';

const props = defineProps({
  open: {
    type: Boolean,
    required: true,
  },
  id: {
    type: String,
    required: true,
  },
  tree: {
    type: Object as PropType<MaybePartialLineageWithMetadata>,
    required: true,
  },
});

const emit = defineEmits<{
  (e: 'close'): void;
}>();

const field = ref();
const error = ref();
const isLoadedAndOk = ref(false);
const problemDragon = ref<PartialLineage>();
const status = ref<InstanceType<typeof Feedback>>();

const viewLink = ref('');

function reset() {
  isLoadedAndOk.value = false;
  problemDragon.value = undefined;
  viewLink.value = '';
}

onUpdated(async () => {
  if (!status.value) return;

  reset();

  const incomingTree = Lineage(props.tree);

  try {
    status.value.info('Attempting to save lineage...');

    const link = await incomingTree.saveToServer();

    status.value.close(() => {
      isLoadedAndOk.value = true;
      viewLink.value = link;
    });
  } catch (ex) {
    let message = '';

    if (ex instanceof ValidationError) {
      field.value = ex.type;
      error.value = ex.message;

      if (ex.type !== 'generation-count') {
        problemDragon.value = incomingTree
          .getAtPath(
            (ex.path as string).substring(0, ex.path?.lastIndexOf('.')),
          )
          ?.raw();
        message = 'The problem dragon is displayed below.';
      } else {
        message = ex.message;
      }
    } else if (ex instanceof FetchError) {
      message = `You may want to try again or export it instead.`;
    }

    status.value.error(
      `Sorry, an error has occurred while
            saving the lineage.<br /> ${message}`,
    );
  }
});
</script>
