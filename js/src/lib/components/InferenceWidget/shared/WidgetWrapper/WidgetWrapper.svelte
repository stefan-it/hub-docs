<script lang="ts">
	import type { WidgetProps, ModelLoadInfo } from "../types";
	import type { WidgetExample } from "../WidgetExample";

	type TWidgetExample = $$Generic<WidgetExample>;

	import { onMount } from "svelte";

	import IconCross from "../../../Icons/IconCross.svelte";
	import WidgetInputSamples from "../WidgetInputSamples/WidgetInputSamples.svelte";
	import WidgetInputSamplesGroup from "../WidgetInputSamplesGroup/WidgetInputSamplesGroup.svelte";
	import WidgetFooter from "../WidgetFooter/WidgetFooter.svelte";
	import WidgetHeader from "../WidgetHeader/WidgetHeader.svelte";
	import WidgetInfo from "../WidgetInfo/WidgetInfo.svelte";
	import WidgetModelLoading from "../WidgetModelLoading/WidgetModelLoading.svelte";
	import { getModelLoadInfo } from "../../shared/helpers";
	import { modelLoadStates } from "../../stores";

	export let apiUrl: string;
	export let computeTime: string;
	export let error: string;
	export let isLoading = false;
	export let model: WidgetProps["model"];
	export let includeCredentials: WidgetProps["includeCredentials"];
	export let modelLoading = {
		isLoading: false,
		estimatedTime: 0,
	};
	export let noTitle = false;
	export let outputJson: string;
	export let applyInputSample: (sample: TWidgetExample) => void = () => {};
	export let previewInputSample: (sample: TWidgetExample) => void = () => {};
	export let validateExample: (sample: WidgetExample) => sample is TWidgetExample;

	let isMaximized = false;
	let modelLoadInfo: ModelLoadInfo | undefined = undefined;
	let selectedInputGroup: string;

	const inputSamples = (model.widgetData ?? [])
		.filter(validateExample)
		.sort((sample1, sample2) => (sample2.example_title ? 1 : 0) - (sample1.example_title ? 1 : 0))
		.map((sample, idx) => ({
			example_title: `Example ${++idx}`,
			group: "Group 1",
			...sample,
		}));

	const inputGroups: {
		group: string;
		inputSamples: TWidgetExample[];
	}[] = [];
	for (const inputSample of inputSamples) {
		const isExist = inputGroups.find(({ group }) => group === inputSample.group);
		if (!isExist) {
			inputGroups.push({ group: inputSample.group as string, inputSamples: [] });
		}
		inputGroups.find(({ group }) => group === inputSample.group)?.inputSamples.push(inputSample);
	}

	$: selectedInputSamples =
		inputGroups.length === 1 ? inputGroups[0] : inputGroups.find(({ group }) => group === selectedInputGroup);

	onMount(() => {
		(async () => {
			modelLoadInfo = await getModelLoadInfo(apiUrl, model.id, includeCredentials);
			$modelLoadStates[model.id] = modelLoadInfo;
		})();
	});

	function onClickMaximizeBtn() {
		isMaximized = !isMaximized;
	}
</script>

<div
	class="flex w-full max-w-full flex-col
	{isMaximized ? 'fixed inset-0 z-20 bg-white p-12' : ''}
	{!modelLoadInfo ? 'hidden' : ''}"
>
	{#if modelLoadInfo?.state === "TooBig"}
		<p class="text-sm text-gray-500">
			Model is supported, but too large to load onto the free Inference API. To try the model, launch it on <a
				class="underline"
				href="https://ui.endpoints.huggingface.co/new?repository={encodeURIComponent(model.id)}">Inference Endpoints</a
			>
			instead.
		</p>
	{:else}
		{#if isMaximized}
			<button class="absolute right-12 top-6" on:click={onClickMaximizeBtn}>
				<IconCross classNames="text-xl text-gray-500 hover:text-black" />
			</button>
		{/if}
		<WidgetHeader {noTitle} pipeline={model.pipeline_tag}>
			{#if !!inputGroups.length}
				<div class="ml-auto flex gap-x-1">
					<!-- Show samples selector when there are more than one sample -->
					{#if inputGroups.length > 1}
						<WidgetInputSamplesGroup
							bind:selectedInputGroup
							{isLoading}
							inputGroups={inputGroups.map(({ group }) => group)}
						/>
					{/if}
					<WidgetInputSamples
						classNames={!selectedInputSamples ? "opacity-50 pointer-events-none" : ""}
						{isLoading}
						inputSamples={selectedInputSamples?.inputSamples ?? []}
						{applyInputSample}
						{previewInputSample}
					/>
				</div>
			{/if}
		</WidgetHeader>
		<slot name="top" />
		<WidgetInfo {model} {computeTime} {error} {modelLoadInfo} />
		{#if modelLoading.isLoading}
			<WidgetModelLoading estimatedTime={modelLoading.estimatedTime} />
		{/if}
		<slot name="bottom" />
		<WidgetFooter {onClickMaximizeBtn} {outputJson} />
	{/if}
</div>
