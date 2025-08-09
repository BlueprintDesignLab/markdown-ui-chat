<script lang="ts">
	import OpenAI from 'openai';

    import { MarkdownUI } from '@markdown-ui/svelte';
    import '@markdown-ui/svelte/widgets.css'; // Optional styling

    import { Marked } from 'marked';
    import { markedUiExtension } from "@markdown-ui/marked-ext";

    // Setup marked with the markdown-ui extension
    const marked = new Marked();
    marked.use(markedUiExtension);

    function handleWidgetEvent(detail: any) {
        console.log('User selected:', detail); 

        inputMessage = JSON.stringify(detail.value);
        sendMessage();
        // Output: {id: "env", value: "prod"}
    }

	interface ChatMessage {
		id: string;
		role: 'user' | 'assistant' | 'system';
		content: string;
		timestamp: Date;
	}

    const initSystemMessage = `
You are an expert demonstrator for markdown-ui. 
Use every opportunity to show the usefulness of markdown-ui.
At the start prefer simple, non text-input widgets as they require less effort for the user.
If the user has engaged for 3+ turns, show them a simple form widget.
Your interaction style is always concise. Max 3 sentences.

You can embed interactive UI widgets in Markdown using fenced code blocks with language "markdown-ui-widget". Inside each code block, output a single JSON object that defines the widget. Do not include any extra prose inside the code block. Keep JSON strictly valid (double quotes, no trailing commas).

Supported widgets and their schemas:

1. textInput { "type": "textInput", "label": string?, // optional "placeholder": string?, // optional "default": string? // optional }

2. buttonGroup { "type": "buttonGroup", "label": string?, // optional "choices": string[], // required "default": string? // optional, must be one of choices }

3. select { "type": "select", "label": string?, // optional "choices": string[], // required "default": string? // optional, must be one of choices }

4. selectMulti { "type": "selectMulti", "label": string?, // optional "choices": string[], // required "default": string | string[]? // optional, must be subset of choices }

5. slider { "type": "slider", "label": string?, // optional "min": number, // required "max": number, // required "step": number?, // optional (default 1) "default": number? // optional, within [min, max] }

6. form { "type": "form", "submitLabel": string?, // optional "fields": Field[] // required } Where Field is any of: textInput | buttonGroup | select | selectMulti | slider, and each field object MUST include: { "type": "...", // one of the above "id": string, // required, unique within the form ...other props per the chosen field type (see schemas above) }

Output rules:

- Use one widget per fenced code block. Only one widget per response.
- Keep all surrounding content as normal Markdown prose outside the widget code fences.
- Only use the widget types and properties listed above; do not invent new types or props.
- Keep JSON minimal; omit optional properties unless they add value.
- Do not include comments inside JSON.
`

	let messages: ChatMessage[] = [{
        id: crypto.randomUUID(),
        role: "system",
        content: initSystemMessage,
        timestamp: new Date(),
    }];
	let inputMessage = '';
	let isLoading = false;
	let isStreaming = false;
	let apiKey = '';
	let showApiKeyInput = true;
	let showSystemMessage = false;
	let openai: OpenAI | null = null;
	let streamingMessageId = '';
	let messagesContainer: HTMLDivElement;

	function scrollToBottom() {
		if (messagesContainer) {
			requestAnimationFrame(() => {
				messagesContainer.scrollTop = messagesContainer.scrollHeight;
			});
		}
	}

	// Autoscroll when messages change or during streaming
	$: if (messages.length > 0 || isStreaming) {
		scrollToBottom();
	}

	function updateSystemMessage(content: string) {
		messages = messages.map(msg => 
			msg.role === 'system' 
				? { ...msg, content }
				: msg
		);
	}

	function getSystemMessage(): string {
		return messages.find(msg => msg.role === 'system')?.content || '';
	}

	function initializeOpenAI() {
		if (apiKey.trim()) {
			openai = new OpenAI({
				apiKey: apiKey,
				dangerouslyAllowBrowser: true
			});
		}
	}

	async function sendMessage() {
		if (!inputMessage.trim() || !apiKey.trim() || isLoading || isStreaming || !openai) return;

		const userMessage: ChatMessage = {
			id: crypto.randomUUID(),
			role: 'user',
			content: inputMessage.trim(),
			timestamp: new Date()
		};

		messages = [...messages, userMessage];
		inputMessage = '';
		isLoading = true;

		// Create a placeholder message for the streaming response
		const assistantMessageId = crypto.randomUUID();
		const assistantMessage: ChatMessage = {
			id: assistantMessageId,
			role: 'assistant',
			content: '',
			timestamp: new Date()
		};

		messages = [...messages, assistantMessage];
		streamingMessageId = assistantMessageId;
		isLoading = false;
		isStreaming = true;

		try {
			const stream = await openai.chat.completions.create({
				model: 'gpt-5-nano',
				messages: messages.filter(m => m.id !== assistantMessageId).map(m => ({
					role: m.role,
					content: m.content
				})),
				stream: true
			});

			let streamedContent = '';

			for await (const chunk of stream) {
				const delta = chunk.choices[0]?.delta?.content || '';
				if (delta) {
					streamedContent += delta;
					
					// Update the message in the messages array
					messages = messages.map(msg => 
						msg.id === assistantMessageId 
							? { ...msg, content: streamedContent }
							: msg
					);
				}
			}

		} catch (error) {
			console.error('Error sending message:', error);
			const errorContent = `Error: ${error instanceof Error ? error.message : 'Failed to send message'}`;
			
			// Update the streaming message with error content
			messages = messages.map(msg => 
				msg.id === assistantMessageId 
					? { ...msg, content: errorContent }
					: msg
			);
		} finally {
			isStreaming = false;
			streamingMessageId = '';
		}
	}

	function handleKeyPress(event: KeyboardEvent) {
		if (event.key === 'Enter' && !event.shiftKey) {
			event.preventDefault();
			sendMessage();
		}
	}

	function saveApiKey() {
		if (apiKey.trim()) {
			showApiKeyInput = false;
			localStorage.setItem('openai_api_key', apiKey);
			initializeOpenAI();
		}
	}

	function clearChat() {
		messages = messages.filter(m => m.role === 'system');
	}

	// Load API key from localStorage on mount
	import { onMount } from 'svelte';
	onMount(() => {
		const savedApiKey = localStorage.getItem('openai_api_key');
		if (savedApiKey) {
			apiKey = savedApiKey;
			showApiKeyInput = false;
			initializeOpenAI();
		}
	});
</script>

<div class="min-h-screen bg-gray-50 flex flex-col">
	<!-- Header -->
	<header class="bg-white shadow-sm border-b border-gray-200 px-4 py-3">
		<div class="max-w-4xl mx-auto flex items-center justify-between">
			<h1 class="text-xl font-semibold text-gray-900">Markdown UI Chat</h1>
			<div class="flex items-center gap-2">
				<a
					target="_blank"
					rel="noopener noreferrer"
					href="https://github.com/BlueprintDesignLab/markdown-ui/"
					class="text-sm px-3 py-1 bg-gray-100 text-gray-700 rounded-md hover:bg-gray-200 transition-colors"
				>
					Markdown UI Github
				</a>
				<button
					on:click={() => showSystemMessage = true}
					class="text-sm px-3 py-1 bg-purple-100 text-purple-700 rounded-md hover:bg-purple-200 transition-colors"
				>
					System Message
				</button>
				<button
					on:click={() => showApiKeyInput = true}
					class="text-sm px-3 py-1 bg-gray-100 text-gray-700 rounded-md hover:bg-gray-200 transition-colors"
				>
					{apiKey ? 'Change API Key' : 'Set API Key'}
				</button>
                
				<button
					on:click={clearChat}
					class="text-sm px-3 py-1 bg-red-100 text-red-700 rounded-md hover:bg-red-200 transition-colors"
				>
					Clear Chat
				</button>
			</div>
		</div>
	</header>

	<!-- API Key Input Modal -->
	{#if showApiKeyInput}
		<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
			<div class="bg-white rounded-lg p-6 w-full max-w-md mx-4">
				<h2 class="text-lg font-semibold mb-4">Enter OpenAI API Key</h2>
				<input
					bind:value={apiKey}
					type="password"
					placeholder="sk-..."
					class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
				>
				<div class="flex gap-2 mt-4">
					<button
						on:click={saveApiKey}
						class="flex-1 px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors disabled:opacity-50"
						disabled={!apiKey.trim()}
					>
						Save
					</button>
					<button
						on:click={() => showApiKeyInput = false}
						class="px-4 py-2 border border-gray-300 text-gray-700 rounded-md hover:bg-gray-50 transition-colors"
					>
						Cancel
					</button>
				</div>
			</div>
		</div>
	{/if}

	<!-- System Message Input Modal -->
	{#if showSystemMessage}
		<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
			<div class="bg-white rounded-lg p-6 w-full max-w-4xl mx-4">
				<h2 class="text-lg font-semibold mb-4">Edit System Message</h2>
				<textarea
					value={getSystemMessage()}
					on:input={(e) => updateSystemMessage((e.target as HTMLTextAreaElement).value)}
					placeholder="Enter system message (e.g., 'You are a helpful AI assistant.')"
					class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 resize-vertical"
					rows="12"
				></textarea>
				<div class="flex gap-2 mt-4">
					<button
						on:click={() => showSystemMessage = false}
						class="flex-1 px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors"
					>
						Save
					</button>
					<button
						on:click={() => showSystemMessage = false}
						class="px-4 py-2 border border-gray-300 text-gray-700 rounded-md hover:bg-gray-50 transition-colors"
					>
						Cancel
					</button>
				</div>
			</div>
		</div>
	{/if}

	<!-- Main Content -->
	<main class="flex-1 flex flex-col max-w-4xl mx-auto w-full pb-24">
		<!-- Messages Container -->
		<div bind:this={messagesContainer} class="flex-1 overflow-y-auto px-4 py-4">
			{#if messages.filter(m => m.role !== 'system').length === 0}
				<div class="text-center text-gray-500 mt-8">
					<p class="text-lg mb-2">Welcome to ChatGPT!</p>
					<p class="text-sm">Start a conversation by typing a message below.</p>
				</div>
			{/if}

			{#each messages.filter(m => m.role !== 'system') as message (message.id)}
				<div class="mb-6 flex {message.role === 'user' ? 'justify-end' : 'justify-start'}">
					<div class="flex max-w-xs lg:max-w-2xl {message.role === 'user' ? 'flex-row-reverse' : 'flex-row'} gap-3">
						<!-- Avatar -->
						<div class="flex-shrink-0">
							<div class="w-8 h-8 rounded-full flex items-center justify-center {message.role === 'user' ? 'bg-blue-600 text-white' : 'bg-green-600 text-white'}">
								{message.role === 'user' ? 'U' : 'AI'}
							</div>
						</div>
						
						<!-- Message Bubble -->
						<div class="flex flex-col {message.role === 'user' ? 'items-end' : 'items-start'}">
							<div class="px-4 py-2 rounded-lg {message.role === 'user' ? 'bg-blue-600 text-white' : 'bg-white text-gray-900 shadow-sm border border-gray-200'} relative">
								<MarkdownUI 
                                    html={marked.parse(message.content)} 
                                    onwidgetevent={handleWidgetEvent} 
                                />
								
								<!-- Streaming indicator -->
								{#if isStreaming && message.id === streamingMessageId}
									<div class="flex items-center gap-1 mt-2">
										<div class="flex items-center gap-1">
											<div class="w-1.5 h-1.5 bg-blue-500 rounded-full animate-pulse"></div>
											<div class="w-1.5 h-1.5 bg-blue-500 rounded-full animate-pulse" style="animation-delay: 0.2s"></div>
											<div class="w-1.5 h-1.5 bg-blue-500 rounded-full animate-pulse" style="animation-delay: 0.4s"></div>
										</div>
										<span class="text-xs text-gray-500 ml-2">AI is typing...</span>
									</div>
								{/if}
							</div>
							<div class="text-xs text-gray-500 mt-1 px-2">
								{message.timestamp.toLocaleTimeString()}
								{#if isStreaming && message.id === streamingMessageId}
									<span class="ml-2 text-blue-500">● Streaming</span>
								{/if}
							</div>
						</div>
					</div>
				</div>
			{/each}

		</div>
	</main>

	<!-- Floating Input Area -->
	<div class="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 px-4 py-4 z-40">
		<div class="max-w-4xl mx-auto">
			<div class="flex gap-2">
				<textarea
					bind:value={inputMessage}
					on:keydown={handleKeyPress}
					placeholder="Type your message here... (Press Enter to send, Shift+Enter for new line)"
					class="flex-1 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 resize-none"
					rows="1"
					disabled={!apiKey.trim() || isLoading || isStreaming}
				></textarea>
				<button
					on:click={sendMessage}
					disabled={!inputMessage.trim() || !apiKey.trim() || isLoading || isStreaming}
					class="px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors disabled:opacity-50 disabled:cursor-not-allowed flex items-center gap-2"
				>
					{#if isLoading}
						<div class="w-4 h-4 border-2 border-white border-t-transparent rounded-full animate-spin"></div>
					{:else if isStreaming}
						<div class="flex items-center gap-1">
							<div class="w-2 h-2 bg-white rounded-full animate-bounce"></div>
							<div class="w-2 h-2 bg-white rounded-full animate-bounce" style="animation-delay: 0.1s"></div>
							<div class="w-2 h-2 bg-white rounded-full animate-bounce" style="animation-delay: 0.2s"></div>
						</div>
					{/if}
					{isStreaming ? 'Streaming...' : 'Send'}
				</button>
			</div>
		</div>
	</div>
</div>
