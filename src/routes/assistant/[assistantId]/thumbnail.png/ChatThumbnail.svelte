<script lang="ts">
	import { page } from "$app/stores";
	import { env as envPublic } from "$env/dynamic/public";
	import { base } from "$app/paths";

	interface Props {
		name: string;
		description?: string;
		createdByName: string | undefined;
		avatar: string | undefined;
	}

	let { name, description = "", createdByName, avatar }: Props = $props();
</script>

<div class="flex h-full w-full flex-col items-center justify-center bg-black p-2">
	<div class="flex w-full max-w-[540px] items-start justify-center text-white">
		{#if avatar}
			<img class="h-64 w-64 rounded-full" style="object-fit: cover;" src={avatar} alt="avatar" />
		{/if}
		<div class="ml-10 flex flex-col items-start">
			<p class="mb-2 mt-0 text-3xl font-normal text-gray-400">
				<span class="mr-1.5 h-8 w-8">
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
				</span>
				AI assistant
			</p>
			<h1 class="m-0 {name.length < 38 ? 'text-5xl' : 'text-4xl'} font-black">
				{name}
			</h1>
			<p class="mb-8 text-2xl">
				{description.slice(0, 160)}
				{#if description.length > 160}...{/if}
			</p>
			<div class="rounded-full bg-[#FFA800] px-8 py-3 text-3xl font-semibold text-black">
				Start chatting
			</div>
		</div>
	</div>
	{#if createdByName}
		<p class="absolute bottom-4 right-8 text-2xl text-gray-400">
			An AI assistant created by {createdByName}
		</p>
	{/if}
</div>
