<script lang="ts">
  import Combobox, { type ComboBoxOption } from '$lib/components/shared-components/combobox.svelte';
  import { t } from 'svelte-i18n';
  import type { Snippet } from 'svelte';

  interface Props {
    label: string;
    comboboxPlaceholder: string;
    subtitle?: string;
    options: ComboBoxOption[];
    selectedOption: {value: string, label: string};
    onSelect: (combobox: ComboBoxOption | undefined) => void;
    children?: Snippet;
  }

  let {
    label,
    comboboxPlaceholder,
    options,
    selectedOption,
    onSelect,
    children,
  }: Props = $props();
</script>

<div class="grid grid-cols-5 items-center w-full">
  <label class="font-medium text-immich-primary dark:text-immich-dark-primary text-sm col-span-1 text-center ml-2">
    {label}
  </label>
  <div class="col-span-4 pr-2">
    <Combobox label='' hideLabel={true} {selectedOption} {options} placeholder={comboboxPlaceholder} {onSelect} />
    {@render children?.()}
  </div>
</div>
