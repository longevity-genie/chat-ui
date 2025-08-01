<script lang="ts">
	import { createBubbler } from "svelte/legacy";

	const bubble = createBubbler();
	import type { Message, MessageFile } from "$lib/types/Message";
	import { createEventDispatcher, onDestroy, tick } from "svelte";

	import CarbonExport from "~icons/carbon/export";
	import CarbonCheckmark from "~icons/carbon/checkmark";
	import CarbonCaretDown from "~icons/carbon/caret-down";

	import EosIconsLoading from "~icons/eos-icons/loading";

	import ChatInput from "./ChatInput.svelte";
	import StopGeneratingBtn from "../StopGeneratingBtn.svelte";
	import type { Model } from "$lib/types/Model";
	import { page } from "$app/state";
	import FileDropzone from "./FileDropzone.svelte";
	import RetryBtn from "../RetryBtn.svelte";
	import file2base64 from "$lib/utils/file2base64";
	import type { Assistant } from "$lib/types/Assistant";
	import { base } from "$app/paths";
	import ContinueBtn from "../ContinueBtn.svelte";
	import AssistantIntroduction from "./AssistantIntroduction.svelte";
	import ChatMessage from "./ChatMessage.svelte";
	import ScrollToBottomBtn from "../ScrollToBottomBtn.svelte";
	import ScrollToPreviousBtn from "../ScrollToPreviousBtn.svelte";
	import { browser } from "$app/environment";
	import { snapScrollToBottom } from "$lib/actions/snapScrollToBottom";
	import SystemPromptModal from "../SystemPromptModal.svelte";
	import ChatIntroduction from "./ChatIntroduction.svelte";
	import UploadedFile from "./UploadedFile.svelte";
	import { useSettingsStore } from "$lib/stores/settings";
	import ModelSwitch from "./ModelSwitch.svelte";

	import { fly } from "svelte/transition";
	import { cubicInOut } from "svelte/easing";
	import type { ToolFront } from "$lib/types/Tool";
	import { loginModalOpen } from "$lib/stores/loginModal";
	import { beforeNavigate } from "$app/navigation";
	import { isVirtualKeyboard } from "$lib/utils/isVirtualKeyboard";

	interface Props {
		messages?: Message[];
		messagesAlternatives?: Message["id"][][];
		loading?: boolean;
		pending?: boolean;
		shared?: boolean;
		currentModel: Model;
		models: Model[];
		assistant?: Assistant | undefined;
		preprompt?: string | undefined;
		files?: File[];
	}

	let {
		messages = [],
		messagesAlternatives = [],
		loading = false,
		pending = false,
		shared = false,
		currentModel,
		models,
		assistant = undefined,
		preprompt = undefined,
		files = $bindable([]),
	}: Props = $props();

	let isReadOnly = $derived(!models.some((model) => model.id === currentModel.id));

	let message: string = $state("");
	let timeout: ReturnType<typeof setTimeout>;
	let isSharedRecently = $state(false);
	let editMsdgId: Message["id"] | null = $state(null);
	let pastedLongContent = $state(false);

	beforeNavigate(() => {
		if (page.params.id) {
			isSharedRecently = false;
		}
	});

	const dispatch = createEventDispatcher<{
		message: string;
		share: void;
		stop: void;
		retry: { id: Message["id"]; content?: string };
		continue: { id: Message["id"] };
	}>();

	const handleSubmit = () => {
		if (loading) return;
		dispatch("message", message);
		message = "";
	};

	let lastTarget: EventTarget | null = null;

	let onDrag = $state(false);

	const onDragEnter = (e: DragEvent) => {
		lastTarget = e.target;
		onDrag = true;
	};
	const onDragLeave = (e: DragEvent) => {
		if (e.target === lastTarget) {
			onDrag = false;
		}
	};

	const onPaste = (e: ClipboardEvent) => {
		const textContent = e.clipboardData?.getData("text");

		if (!$settings.directPaste && textContent && textContent.length >= 3984) {
			e.preventDefault();
			pastedLongContent = true;
			setTimeout(() => {
				pastedLongContent = false;
			}, 1000);
			const pastedFile = new File([textContent], "Pasted Content", {
				type: "application/vnd.chatui.clipboard",
			});

			files = [...files, pastedFile];
		}

		if (!e.clipboardData) {
			return;
		}

		// paste of files
		const pastedFiles = Array.from(e.clipboardData.files);
		if (pastedFiles.length !== 0) {
			e.preventDefault();

			// filter based on activeMimeTypes, including wildcards
			const filteredFiles = pastedFiles.filter((file) => {
				return activeMimeTypes.some((mimeType: string) => {
					const [type, subtype] = mimeType.split("/");
					const [fileType, fileSubtype] = file.type.split("/");
					return (
						(type === "*" || fileType === type) && (subtype === "*" || fileSubtype === subtype)
					);
				});
			});

			files = [...files, ...filteredFiles];
		}
	};

	let lastMessage = $derived(browser && (messages.at(-1) as Message));
	let lastIsError = $derived(
		lastMessage &&
			!loading &&
			(lastMessage.from === "user" ||
				lastMessage.updates?.findIndex((u) => u.type === "status" && u.status === "error") !== -1)
	);

	let sources = $derived(
		files?.map<Promise<MessageFile>>((file) =>
			file2base64(file).then((value) => ({
				type: "base64",
				value,
				mime: file.type,
				name: file.name,
			}))
		)
	);

	function onShare() {
		if (!confirm("Are you sure you want to share this conversation? This cannot be undone.")) {
			return;
		}

		dispatch("share");
		isSharedRecently = true;
		if (timeout) {
			clearTimeout(timeout);
		}
		timeout = setTimeout(() => {
			isSharedRecently = false;
		}, 2000);
	}

	onDestroy(() => {
		if (timeout) {
			clearTimeout(timeout);
		}
	});

	let chatContainer: HTMLElement | undefined = $state();

	async function scrollToBottom() {
		await tick();
		if (!chatContainer) return;
		chatContainer.scrollTop = chatContainer.scrollHeight;
	}

	// If last message is from user, scroll to bottom
	$effect(() => {
		if (lastMessage && lastMessage.from === "user") {
			scrollToBottom();
		}
	});

	const settings = useSettingsStore();

  // Old Svelte Syntax

	// $: mimeTypesFromActiveTools = $page.data.tools
	// 	.filter((tool: ToolFront) => {
	// 		if ($page.data?.assistant) {
	// 			return $page.data.assistant.tools?.includes(tool._id);
	// 		}
	// 		if (currentModel.tools) {
	// 			return $settings?.tools?.includes(tool._id) ?? tool.isOnByDefault;
	// 		}
	// 		return false;
	// 	})
	// 	.flatMap((tool: ToolFront) => tool.mimeTypes ?? []);
	//
	// $: activeMimeTypes = Array.from(
	// 	new Set([
	// 		...mimeTypesFromActiveTools, // fetch mime types from active tools either from tool settings or active assistant
	// 		...(currentModel.tools && !$page.data.assistant ? ["application/pdf"] : []), // if its a tool model, we can always enable document parser so we always accept pdfs
	// 		...(currentModel.multimodal ? currentModel.multimodalAcceptedMimetypes ?? ["*"] : []), // if its a multimodal model, we always accept images ( [lg] changed to accept all mime types for LongevityGenie project)	
	// 	])

	let mimeTypesFromActiveTools = $derived(
		page.data.tools
			.filter((tool: ToolFront) => {
				if (assistant) {
					return assistant.tools?.includes(tool._id);
				}
				if (currentModel.tools) {
					return $settings?.tools?.includes(tool._id) ?? tool.isOnByDefault;
				}
				return false;
			})
			.flatMap((tool: ToolFront) => tool.mimeTypes ?? [])
	);

	let activeMimeTypes = $derived(
		Array.from(
			new Set([
				...mimeTypesFromActiveTools, // fetch mime types from active tools either from tool settings or active assistant
				...(currentModel.tools && !assistant ? ["application/pdf"] : []), // if its a tool model, we can always enable document parser so we always accept pdfs
				...(currentModel.multimodal
					? (currentModel.multimodalAcceptedMimetypes ?? ["*"])
					: []), // if its a multimodal model, we always accept images ( [lg] changed to accept all mime types for LongevityGenie project)
			])
		)
	);
	let isFileUploadEnabled = $derived(activeMimeTypes.length > 0);
	let focused = $state(false);
</script>

<svelte:window
	ondragenter={onDragEnter}
	ondragleave={onDragLeave}
	ondragover={(e) => {
		e.preventDefault();
		bubble("dragover");
	}}
	ondrop={(e) => {
		e.preventDefault();
		onDrag = false;
	}}
/>

<div class="relative z-[-1] min-h-0 min-w-0">
	<div
		class="scrollbar-custom h-full overflow-y-auto"
		use:snapScrollToBottom={messages.map((message) => message.content)}
		bind:this={chatContainer}
	>
		<div
			class="mx-auto flex h-full max-w-3xl flex-col gap-6 px-5 pt-6 sm:gap-8 xl:max-w-4xl xl:pt-10"
		>
			{#if assistant && !!messages.length}
				<a
					class="mx-auto flex items-center gap-1.5 rounded-full border border-gray-100 bg-gray-50 py-1 pl-1 pr-3 text-sm text-gray-800 hover:bg-gray-100 dark:border-gray-800 dark:bg-gray-800 dark:text-gray-200 dark:hover:bg-gray-700"
					href="{base}/assistant/{assistant._id}"
				>
					{#if assistant.avatar}
						<img
							src="{base}/settings/assistants/{assistant._id.toString()}/avatar.jpg?hash=${assistant.avatar}"
							alt="Avatar"
							class="size-5 rounded-full object-cover"
						/>
					{:else}
						<div
							class="flex size-6 items-center justify-center rounded-full bg-gray-300 font-bold uppercase text-gray-500"
						>
							{assistant.name[0]}
						</div>
					{/if}

					{assistant.name}
				</a>
			{:else if preprompt && preprompt != currentModel.preprompt}
				<SystemPromptModal preprompt={preprompt ?? ""} />
			{/if}

			{#if messages.length > 0}
				<div class="flex h-max flex-col gap-8 pb-52">
					{#each messages as message, idx (message.id)}
						<ChatMessage
							{loading}
							{message}
							alternatives={messagesAlternatives.find((a) => a.includes(message.id)) ?? []}
							isAuthor={!shared}
							readOnly={isReadOnly}
							isLast={idx === messages.length - 1}
							bind:editMsdgId
							on:retry
							on:vote
							on:continue
							on:showAlternateMsg
						/>
					{/each}
					{#if isReadOnly}
						<ModelSwitch {models} {currentModel} />
					{/if}
				</div>
			{:else if pending}
				<ChatMessage
					loading={true}
					message={{
						id: "0-0-0-0-0",
						content: "",
						from: "assistant",
						children: [],
					}}
					isAuthor={!shared}
					readOnly={isReadOnly}
				/>
			{:else if !assistant}
				<ChatIntroduction
					{currentModel}
					on:message={(ev) => {
						if (page.data.loginRequired) {
							ev.preventDefault();
							$loginModalOpen = true;
						} else {
							dispatch("message", ev.detail);
						}
					}}
				/>
			{:else}
				<AssistantIntroduction
					{models}
					{assistant}
					on:message={(ev) => {
						if (page.data.loginRequired) {
							ev.preventDefault();
							$loginModalOpen = true;
						} else {
							dispatch("message", ev.detail);
						}
					}}
				/>
			{/if}
		</div>

		<ScrollToPreviousBtn
			class="fixed right-4 max-md:bottom-[calc(50%+26px)] md:bottom-48 lg:right-10"
			scrollNode={chatContainer}
		/>

		<ScrollToBottomBtn
			class="fixed right-4 max-md:bottom-[calc(50%-26px)] md:bottom-36 lg:right-10"
			scrollNode={chatContainer}
		/>
	</div>
	<div
		class="pointer-events-none absolute inset-x-0 bottom-0 z-0 mx-auto flex w-full
			max-w-3xl flex-col items-center justify-center bg-gradient-to-t from-white
			via-white/100 to-white/0 px-3.5 pt-2 dark:border-gray-800
			dark:from-gray-900 dark:via-gray-900/100
			dark:to-gray-900/0 max-sm:py-0 sm:px-5 md:pb-4 xl:max-w-4xl [&>*]:pointer-events-auto"
	>
		{#if sources?.length && !loading}
			<div
				in:fly|local={sources.length === 1 ? { y: -20, easing: cubicInOut } : undefined}
				class="flex flex-row flex-wrap justify-center gap-2.5 rounded-xl pb-3"
			>
				{#each sources as source, index}
					{#await source then src}
						<UploadedFile
							file={src}
							on:close={() => {
								files = files.filter((_, i) => i !== index);
							}}
						/>
					{/await}
				{/each}
			</div>
		{/if}

		<div class="w-full">
			<div class="flex w-full *:mb-3">
				{#if loading}
					<StopGeneratingBtn classNames="ml-auto" onClick={() => dispatch("stop")} />
				{:else if lastIsError}
					<RetryBtn
						classNames="ml-auto"
						onClick={() => {
							if (lastMessage && lastMessage.ancestors) {
								dispatch("retry", {
									id: lastMessage.id,
								});
							}
						}}
					/>
				{:else if messages && lastMessage && lastMessage.interrupted && !isReadOnly}
					<div class="ml-auto gap-2">
						<ContinueBtn
							onClick={() => {
								if (lastMessage && lastMessage.ancestors) {
									dispatch("continue", {
										id: lastMessage?.id,
									});
								}
							}}
						/>
					</div>
				{/if}
			</div>
			<form
				tabindex="-1"
				aria-label={isFileUploadEnabled ? "file dropzone" : undefined}
				onsubmit={(e) => {
					e.preventDefault();
					handleSubmit();
				}}
				class={{
					"relative flex w-full max-w-4xl flex-1 items-center rounded-xl border bg-gray-100 dark:border-gray-600 dark:bg-gray-700": true,
					"opacity-30": isReadOnly,
					"max-sm:mb-4": focused && isVirtualKeyboard(),
				}}
			>
				{#if onDrag && isFileUploadEnabled}
					<FileDropzone bind:files bind:onDrag mimeTypes={activeMimeTypes} />
				{:else}
					<div
						class="flex w-full flex-1 rounded-xl border-none bg-transparent"
						class:paste-glow={pastedLongContent}
					>
						{#if lastIsError}
							<ChatInput value="Sorry, something went wrong. Please try again." disabled={true} />
						{:else}
							<ChatInput
								{assistant}
								placeholder={isReadOnly ? "This conversation is read-only." : "Ask anything"}
								{loading}
								bind:value={message}
								bind:files
								mimeTypes={activeMimeTypes}
								on:submit={handleSubmit}
								{onPaste}
								disabled={isReadOnly || lastIsError}
								modelHasTools={currentModel.tools}
								modelIsMultimodal={currentModel.multimodal}
								bind:focused
							/>
						{/if}

						{#if loading}
							<button
								disabled
								class="btn absolute bottom-1 right-0.5 size-10 self-end rounded-lg bg-transparent text-gray-400"
							>
								<EosIconsLoading />
							</button>
						{:else}
							<button
								class="btn absolute bottom-2 right-2 size-7 self-end rounded-full border bg-white text-black shadow transition-none enabled:hover:bg-white enabled:hover:shadow-inner disabled:text-gray-400/50 disabled:opacity-60 dark:border-gray-600 dark:bg-gray-900 dark:text-white dark:hover:enabled:bg-black dark:disabled:text-gray-600/50"
								disabled={!message || isReadOnly}
								type="submit"
								aria-label="Send message"
								name="submit"
							>
								<svg
									width="1em"
									height="1em"
									viewBox="0 0 32 32"
									fill="none"
									xmlns="http://www.w3.org/2000/svg"
								>
									<path
										fill-rule="evenodd"
										clip-rule="evenodd"
										d="M17.0606 4.23197C16.4748 3.64618 15.525 3.64618 14.9393 4.23197L5.68412 13.4871C5.09833 14.0729 5.09833 15.0226 5.68412 15.6084C6.2699 16.1942 7.21965 16.1942 7.80544 15.6084L14.4999 8.91395V26.7074C14.4999 27.5359 15.1715 28.2074 15.9999 28.2074C16.8283 28.2074 17.4999 27.5359 17.4999 26.7074V8.91395L24.1944 15.6084C24.7802 16.1942 25.7299 16.1942 26.3157 15.6084C26.9015 15.0226 26.9015 14.0729 26.3157 13.4871L17.0606 4.23197Z"
										fill="currentColor"
									/>
								</svg>
							</button>
						{/if}
					</div>
				{/if}
			</form>
			<div
				class={{
					"mt-2 flex justify-between self-stretch px-1 text-xs text-gray-400/90 max-md:mb-2 max-sm:gap-2": true,
					"max-sm:hidden": focused && isVirtualKeyboard(),
				}}
			>
				<p>
					Model:
					{#if !assistant}
						{#if models.find((m) => m.id === currentModel.id)}
							<a
								href="{base}/settings/{currentModel.id}"
								class="inline-flex items-center hover:underline"
								>{currentModel.displayName}<CarbonCaretDown class="text-xxs" /></a
							>
						{:else}
							<span class="inline-flex items-center line-through dark:border-gray-700">
								{currentModel.id}
							</span>
						{/if}
					{:else}
						{@const model = models.find((m) => m.id === assistant?.modelId)}
						{#if model}
							<a
								href="{base}/settings/assistants/{assistant._id}"
								class="inline-flex items-center border-b hover:text-gray-600 dark:border-gray-700 dark:hover:text-gray-300"
								>{model?.displayName}<CarbonCaretDown class="text-xxs" /></a
							>
						{:else}
							<span class="inline-flex items-center line-through dark:border-gray-700">
								{currentModel.id}
							</span>
						{/if}
					{/if}
					<span class="max-sm:hidden">·</span><br class="sm:hidden" /> Generated content may be inaccurate
					or false.
				</p>
				{#if messages.length}
					<button
						class="flex flex-none items-center hover:text-gray-400 max-sm:rounded-lg max-sm:bg-gray-50 max-sm:px-2.5 dark:max-sm:bg-gray-800"
						type="button"
						class:hover:underline={!isSharedRecently}
						onclick={onShare}
						disabled={isSharedRecently}
					>
						{#if isSharedRecently}
							<CarbonCheckmark class="text-[.6rem] sm:mr-1.5 sm:text-green-600" />
							<div class="text-green-600 max-sm:hidden">Link copied to clipboard</div>
						{:else}
							<CarbonExport class="sm:text-primary-500 text-[.6rem] sm:mr-1.5" />
							<div class="max-sm:hidden">Share this conversation</div>
						{/if}
					</button>
				{/if}
			</div>
		</div>
	</div>
</div>

<style lang="postcss">
	.paste-glow {
		animation: glow 1s cubic-bezier(0.4, 0, 0.2, 1) forwards;
		will-change: box-shadow;
	}

	@keyframes glow {
		0% {
			box-shadow: 0 0 0 0 rgba(59, 130, 246, 0.8);
		}
		50% {
			box-shadow: 0 0 20px 4px rgba(59, 130, 246, 0.6);
		}
		100% {
			box-shadow: 0 0 0 0 rgba(59, 130, 246, 0);
		}
	}
</style>
