<script lang="ts">
	import { env as envPublic } from "$env/dynamic/public";
	import Logo from "$lib/components/icons/Logo.svelte";
	import { createEventDispatcher, onMount } from "svelte";
	import IconGear from "~icons/bi/gear-fill";
	import type { Model } from "$lib/types/Model";
	import ModelCardMetadata from "../ModelCardMetadata.svelte";
	import { base } from "$app/paths";
	import JSON5 from "json5";

	export let currentModel: Model;

  // [lg] Added prompt loading from .env file
	let var_promptExamples: PromptExample[] = [];
	if (currentModel.promptExamples) {
		var_promptExamples = currentModel.promptExamples;
	}
  // [lg] -

	const announcementBanners = envPublic.PUBLIC_ANNOUNCEMENT_BANNERS
		? JSON5.parse(envPublic.PUBLIC_ANNOUNCEMENT_BANNERS)
		: [];

	const dispatch = createEventDispatcher<{ message: string }>();

  // [lg] Added prompt downloading from server
	interface PromptExample {
		prompt: string;
		title: string;
	}

	let promptExamples: PromptExample[] = [];

    onMount(async () => {
		console.log("var_promptExamples", var_promptExamples);
        try {
			let baseURL = currentModel.apiUrl;
			console.log(currentModel);
            const response = await fetch(`${baseURL}/get_prompt_examples`);
            const examples = await response.json();
            // Set prompt examples for current model if they exist
            promptExamples = examples[currentModel.id] || [];
			if (promptExamples.length > 0) { // [lg] Added prompt loading from .env file
				var_promptExamples =  promptExamples;
			}

        } catch (error) {
            console.error('Failed to fetch prompt examples:', error);
            promptExamples = [];
        }
		console.log("var_promptExamples", var_promptExamples);
    });
  // [lg] -

</script>

<div class="my-auto grid gap-8 lg:grid-cols-3">
	<div class="lg:col-span-1">
		<div>
			<div class="mb-3 flex items-center text-2xl font-semibold">
				<Logo classNames="mr-1 flex-none" />
				{envPublic.PUBLIC_APP_NAME}
				<div
					class="ml-3 flex h-6 items-center rounded-lg border border-gray-100 bg-gray-50 px-2 text-base text-gray-400 dark:border-gray-700/60 dark:bg-gray-800"
				>
					v{envPublic.PUBLIC_VERSION}
				</div>
			</div>
			<p class="text-base text-gray-600 dark:text-gray-400">
				{envPublic.PUBLIC_APP_DESCRIPTION ||
					"Making the community's best AI chat models available to everyone."}
			</p>
		</div>
	</div>
  <!-- [lg] Removed Chat Introduction Model -->

  <!-- [lg] Added just-chat promotion link -->
	<div class="lg:col-span-2 lg:pl-24" >
			<a
				href="https://github.com/longevity-genie/just-chat"
				class="mr-2 flex items-center underline hover:no-underline">
				<strong>Support our project Just-Chat!</strong> 
				</a>Make your LLM agent and chat with it simple and fast!
    <!-- [lg] - -->

		<div class="overflow-hidden rounded-xl border dark:border-gray-800">
			<div class="flex p-3">
				<div>
					<div class="text-sm text-gray-600 dark:text-gray-400">Current Model</div>
					<div class="flex items-center gap-1.5 font-semibold max-sm:text-smd">
						{#if currentModel.logoUrl}
							<img
								class=" overflown aspect-square size-4 rounded border dark:border-gray-700"
								src={currentModel.logoUrl}
								alt=""
							/>
						{:else}
							<div class="size-4 rounded border border-transparent bg-indigo-300 dark:bg-gray-800" />
						{/if}
						{currentModel.displayName}
					</div>
				</div>
				<a
					href="{base}/settings/{currentModel.id}"
					class="btn ml-auto flex h-7 w-7 self-start rounded-full bg-indigo-200 p-1 text-xs hover:bg-indigo-50 dark:border-gray-600 dark:bg-gray-800 dark:hover:bg-gray-600"
					><IconGear /></a
				>
			</div>
		</div>
	</div>


    <div class="lg:col-span-3 lg:mt-6">
        <p class="mb-3 text-gray-600 dark:text-gray-300">Examples</p>
        <div class="grid gap-3 lg:grid-cols-3 lg:gap-5">

            {#each var_promptExamples as example}
                <button
                    type="button"
                    class="rounded-xl border bg-indigo-200 p-3 border-indigo-200 text-gray-600 hover:bg-indigo-50 max-xl:text-sm xl:p-3.5 dark:border-gray-800 dark:bg-gray-800 dark:text-gray-300 dark:hover:bg-gray-700"
                    on:click={() => dispatch("message", example.prompt)}
                >
                    {example.title}
                </button>
            {/each}
        </div>
    </div>
	<div class="h-40 sm:h-24" />
</div>
