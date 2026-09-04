<script lang="ts">
	import { Clock3, Plus, Video, VideoOff } from 'lucide-svelte';
	import type { AssistantState } from '$lib/assistant/assistant.svelte.js';
	import { dismissOnOutside } from '$lib/interaction/dismiss.js';
	import RemoveButton from '$lib/ui/primitives/RemoveButton.svelte';
	import { formatDraftDate, fullDraftDate } from '$lib/ui/drafts/draft-date.js';

	let {
		assistant,
		onConversationEmptied
	}: {
		assistant: AssistantState;
		onConversationEmptied?: () => void;
	} = $props();

	let chatsOpen = $state(false);
	let chatsTrigger = $state<HTMLElement>();
	let deleteChatId = $state<string | undefined>();
	let videoInputOpen = $state(false);
	let videoUrlInput = $state('');

	$effect(() => {
		if (videoInputOpen) {
			videoUrlInput = assistant.videoUrl ?? '';
		}
	});

	function dismissChats(): void {
		if (!chatsOpen) return;
		chatsOpen = false;
		deleteChatId = undefined;
	}

	function dismissVideoInput(): void {
		videoInputOpen = false;
	}

	function openChat(id: string): void {
		chatsOpen = false;
		deleteChatId = undefined;
		void assistant.selectChat(id);
	}

	async function deleteChat(id: string): Promise<void> {
		await assistant.deleteChat(id);
		deleteChatId = undefined;
		if (assistant.chats.length === 0) {
			chatsOpen = false;
			onConversationEmptied?.();
		} else {
			chatsTrigger?.focus();
		}
	}

	function setVideoMode(): void {
		const trimmed = videoUrlInput.trim();
		assistant.setVideoUrl(trimmed ? trimmed : undefined);
		videoInputOpen = false;
	}

	function clearVideoMode(): void {
		videoUrlInput = '';
		assistant.setVideoUrl(undefined);
		videoInputOpen = false;
	}
</script>

<div class="assistant-chat-controls">
	<details
		class="assistant-video-mode"
		bind:open={videoInputOpen}
		{@attach dismissOnOutside(dismissVideoInput)}
	>
		<!-- svelte-ignore a11y_no_redundant_roles -->
		<summary
			class="button--quiet icon-button assistant-chats__trigger"
			role="button"
			aria-label={assistant.videoUrl ? 'Video Mode Enabled' : 'Video Mode'}
			title={assistant.videoUrl ? `Video Mode Enabled (${assistant.videoUrl})` : 'Video Mode'}
			aria-expanded={videoInputOpen}
		>
			{#if assistant.videoUrl}
				<Video aria-hidden="true" size={15} strokeWidth={2.25} />
			{:else}
				<VideoOff aria-hidden="true" size={15} strokeWidth={2.25} />
			{/if}
		</summary>
		<div class="assistant-video-mode__popover">
			<h3 class="assistant-chats__heading">Video Support Mode</h3>
			<p class="assistant-video-mode__description">
				Pass YouTube video URL to Gemini Flash for video-assisted transcription checking.
			</p>
			<form
				onsubmit={(e) => {
					e.preventDefault();
					setVideoMode();
				}}
			>
				<input
					type="url"
					class="assistant-video-mode__input"
					placeholder="https://www.youtube.com/watch?v=…"
					bind:value={videoUrlInput}
				/>
				<div class="assistant-video-mode__actions">
					{#if assistant.videoUrl}
						<button type="button" class="button--quiet" onclick={clearVideoMode}> Clear </button>
					{/if}
					<button type="submit" class="button"> Save URL </button>
				</div>
			</form>
		</div>
	</details>
	<details class="assistant-chats" bind:open={chatsOpen} {@attach dismissOnOutside(dismissChats)}>
		<!-- svelte-ignore a11y_no_redundant_roles -->
		<summary
			class="button--quiet icon-button assistant-chats__trigger"
			role="button"
			aria-label="Conversations"
			title="Conversations"
			aria-expanded={chatsOpen}
			bind:this={chatsTrigger}
		>
			<Clock3 aria-hidden="true" size={15} strokeWidth={2.25} />
		</summary>
		<div class="assistant-chats__popover">
			<h3 class="assistant-chats__heading">Conversations</h3>
			{#if assistant.chats.length === 0}
				<p class="assistant-chats__empty">No saved conversations</p>
			{:else}
				<ul class="assistant-chats__list">
					{#each assistant.chats as chat (chat.id)}
						<li class="list-row" class:current={chat.id === assistant.activeChatId}>
							{#if deleteChatId === chat.id}
								<span class="list-row__action list-row__action--static">
									<span class="list-row__name">{chat.title}</span>
								</span>
							{:else}
								<button
									type="button"
									class="list-row__action"
									aria-current={chat.id === assistant.activeChatId ? 'true' : undefined}
									onclick={() => openChat(chat.id)}
								>
									<span class="list-row__name">{chat.title}</span>
									<time datetime={chat.updatedAt} title={fullDraftDate(chat.updatedAt)}>
										{formatDraftDate(chat.updatedAt)}
									</time>
								</button>
							{/if}
							<div class="list-row__commands">
								<!-- The `RemoveButton` stays the same instance across the arming,
								     as the drafts menu and the roster keep theirs: mounted afresh
								     with `pending` already true, its live region is *born* holding
								     the question, and a region that never changes announces
								     nothing. -->
								<RemoveButton
									subject={chat.title}
									pending={deleteChatId === chat.id}
									onRequest={() => (deleteChatId = chat.id)}
									onCancel={() => (deleteChatId = undefined)}
									onConfirm={() => deleteChat(chat.id)}
								/>
							</div>
						</li>
					{/each}
				</ul>
			{/if}
		</div>
	</details>
	<button
		type="button"
		class="button--quiet icon-button"
		disabled={assistant.busy || assistant.challengePending}
		aria-label="New chat"
		title="New chat"
		onclick={() => void assistant.newChat()}
	>
		<Plus aria-hidden="true" size={15} strokeWidth={2.25} />
	</button>
</div>
