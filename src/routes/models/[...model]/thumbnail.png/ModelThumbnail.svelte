<script lang="ts">
	import { env as envPublic } from "$env/dynamic/public";
	import { page } from "$app/stores";
	import { base } from "$app/paths";

	import { usePublicConfig } from "$lib/utils/PublicConfig.svelte";
	const publicConfig = usePublicConfig();

	interface Props {
		name: string;
		logoUrl: string | undefined;
	}

	let { name, logoUrl }: Props = $props();
</script>

<div class=" flex h-[648px] w-full flex-col items-center bg-white">
	<div class="flex flex-1 flex-col items-center justify-center">
		{#if logoUrl}
			<img class="h-48 w-48" src={logoUrl} alt="avatar" />
		{/if}
		<h1 class="m-0 text-5xl font-bold text-black">
			{name}
		</h1>
	</div>

	<div
		class="flex h-[200px] w-full flex-col items-center justify-center rounded-b-none bg-{publicConfig.PUBLIC_APP_COLOR}-500/10 pb-10 pt-10 text-4xl text-gray-500"
		style="border-radius: 100% 100% 0 0;"
	>
		Try it now
		{#if publicConfig.isHuggingChat}
			on
		{/if}

		{#if publicConfig.isHuggingChat}
			<div class="flex flex-row pt-3 text-5xl font-bold text-black">
				<div class="mr-5 flex items-center justify-center" id="logo">
          <!-- [lg] Updated with custom logo + bugfixes -->

					<!-- eslint-disable-next-line -->
					{#if envPublic.PUBLIC_APP_ASSETS === "chatui"}
						<svg
							height="30"
							width="30"
							viewBox="0 0 30 30"
							xmlns="http://www.w3.org/2000/svg"
						>
							<path
								d="M4.06151 14.1464C4.06151 11.8818 4.9611 9.71004 6.56237 8.10877C8.16364 6.5075 10.3354 5.60791 12.6 5.60791H16.5231C18.6254 5.60791 20.6416 6.44307 22.1282 7.92965C23.6148 9.41624 24.45 11.4325 24.45 13.5348C24.45 15.6372 23.6148 17.6534 22.1282 19.14C20.6416 20.6266 18.6254 21.4618 16.5231 21.4618H7.08459L4.63844 23.8387C4.59547 23.8942 4.53557 23.9343 4.4678 23.9527C4.40004 23.9712 4.32811 23.9671 4.2629 23.941C4.1977 23.9149 4.14276 23.8683 4.10643 23.8082C4.07009 23.7481 4.05432 23.6778 4.06151 23.6079V14.1464Z"
								class="fill-primary-500"
							/>
						</svg>
					{:else}
						<img
							alt="{envPublic.PUBLIC_APP_NAME} logo"
							src="{envPublic.PUBLIC_ORIGIN || $page.url.origin}{base}/{envPublic.PUBLIC_APP_ASSETS}/logo.png"
						/>
					{/if}
          <!-- [lg] - -->
				</div>
				<span>HuggingChat</span>
			</div>
		{/if}
	</div>
</div>
