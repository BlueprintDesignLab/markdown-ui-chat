<script lang="ts">
	import OpenAI from 'openai';
	import { onMount, tick } from 'svelte';

    import { MarkdownUI } from '@markdown-ui/svelte';
    import '@markdown-ui/svelte/widgets.css'; // Optional styling

    import { Marked } from 'marked';
    import { markedUiExtension } from "@markdown-ui/marked-ext";

    // Setup marked with the markdown-ui extension
    const marked = new Marked();
    marked.use(markedUiExtension);

    function handleWidgetEvent(detail: any) {
        inputMessage = JSON.stringify(detail.value);
        sendMessage();
    }

	interface ChatMessage {
		id: string;
		role: 'user' | 'assistant' | 'system';
		content: string;
		timestamp: Date;
		parsedHtml?: string;
	}

    const initSystemMessage = `
You are an expert sales person for markdown-ui. 
Use every opportunity to engage the user and make the conversation interesting.
Always show the usefulness of markdown-ui and include at least 1 widget.
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
- MultiSelect and text input should only be part of Form.
- Keep all surrounding content as normal Markdown prose outside the widget code fences.
- Only use the widget types and properties listed above; do not invent new types or props.
- Keep JSON minimal; omit optional properties unless they add value.
- Do not include comments inside JSON.
`

	let messages = $state<ChatMessage[]>([
        {
            id: crypto.randomUUID(),
            role: "system",
            content: initSystemMessage,
            timestamp: new Date(),
        },
    ]);
    
	let inputMessage = $state('');
	let isLoading = $state(false);
	let isStreaming = $state(false);
	let showSystemMessage = $state(false);
	let openai: OpenAI;
	let streamingMessageId = $state('');
	let messagesContainer: HTMLDivElement | undefined = $state();
	let showSuggestions = $state(true);
	let hasUserInteracted = $state(false);

	// Ensure messages have parsed HTML without creating new objects
	$effect(() => {
		messages.forEach((msg) => {
			if (!msg.parsedHtml) {
				msg.parsedHtml = marked.parse(msg.content) as string;
			}
		});
	});

	function scrollToBottom() {
		if (messagesContainer) {
			tick().then(() => {
                // console.log("scrolling", {
                //     scrollHeight: messagesContainer!.scrollHeight,
                //     offsetHeight: messagesContainer!.offsetHeight,
                //     scrollTop: messagesContainer!.scrollTop
                // });
				messagesContainer!.scrollTo({
					top: messagesContainer!.scrollHeight,
					behavior: 'smooth'
				});
			});
		}
	}

	function updateSystemMessage(content: string) {
		const systemMsg = messages.find(msg => msg.role === 'system');
		if (systemMsg) {
			systemMsg.content = content;
			systemMsg.parsedHtml = undefined; // Clear cache to force re-parse
		}
	}

	function getSystemMessage(): string {
		return messages.find(msg => msg.role === 'system')?.content || '';
	}

	function initializeOpenAI() {
		openai = new OpenAI({
			apiKey: 'dummy-key',
			baseURL: 'http://localhost:3010',
			// baseURL: 'https://llm-proxy-735482512776.us-west1.run.app',
			dangerouslyAllowBrowser: true
		});
	}

	async function sendMessage() {
		if (!inputMessage.trim() || isLoading || isStreaming) return;

		if (!hasUserInteracted) {
			hasUserInteracted = true;
			showSuggestions = false;
		}

		const userMessage: ChatMessage = {
			id: crypto.randomUUID(),
			role: 'user',
			content: inputMessage.trim(),
			timestamp: new Date()
		};

		messages.push(userMessage);
		inputMessage = '';
		isLoading = true;
		scrollToBottom();

		// Create a placeholder message for the streaming response
		const assistantMessageId = crypto.randomUUID();
		const assistantMessage: ChatMessage = {
			id: assistantMessageId,
			role: 'assistant',
			content: '',
			timestamp: new Date()
		};

		messages.push(assistantMessage);
		streamingMessageId = assistantMessageId;
		isLoading = false;
		isStreaming = true;
		scrollToBottom();

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
                    scrollToBottom();
					
					// Update the message in the messages array
					const streamingMsg = messages.find(msg => msg.id === assistantMessageId);
					if (streamingMsg) {
						streamingMsg.content = streamedContent;
						streamingMsg.parsedHtml = undefined; // Clear cache to force re-parse during streaming
					}
				}
			}

		} catch (error) {
			console.error('Error sending message:', error);
			const errorContent = `Error: ${error instanceof Error ? error.message : 'Failed to send message'}`;
			
			// Update the streaming message with error content
			const errorMsg = messages.find(msg => msg.id === assistantMessageId);
			if (errorMsg) {
				errorMsg.content = errorContent;
				errorMsg.parsedHtml = undefined; // Clear cache
			}
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


	function clearChat() {
		// Remove non-system messages by mutating the array
		for (let i = messages.length - 1; i >= 0; i--) {
			if (messages[i].role !== 'system') {
				messages.splice(i, 1);
			}
		}
		showSuggestions = true;
		hasUserInteracted = false;
	}

	function handleSuggestionClick(suggestion: string) {
		inputMessage = suggestion;
		sendMessage();
	}

	// Initialize OpenAI client on mount
	onMount(() => {
		initializeOpenAI();
	});
</script>

<div class="h-screen bg-gray-50 flex flex-col overflow-hidden">
	<!-- Header -->
	<header class="bg-white shadow-sm border-b border-gray-200 px-3 sm:px-4 py-3">
		<div class="max-w-4xl mx-auto">
			<div class="flex items-center justify-between">
				<h1 class="text-lg sm:text-xl font-semibold text-gray-900 truncate">
                    Markdown UI Chat Demo
                </h1>
				<button
					onclick={clearChat}
					class="text-xs sm:text-sm px-2 sm:px-3 py-1 bg-red-100 text-red-700 rounded-md hover:bg-red-200 transition-colors flex-shrink-0"
				>
					Clear
				</button>
			</div>
			
			<!-- Mobile-friendly button row -->
			<div class="flex flex-wrap items-center gap-2 mt-2">
                <a
					target="_blank"
					rel="noopener noreferrer"
					href="https://markdown-ui.blueprintlab.io"
					class="text-xs sm:text-sm px-2 sm:px-3 py-1 bg-gray-100 text-gray-700 rounded-md hover:bg-gray-200 transition-colors"
				>
					Markdown UI
				</a>
				<a
					target="_blank"
					rel="noopener noreferrer"
					href="https://github.com/BlueprintDesignLab/markdown-ui/"
					class="text-xs sm:text-sm px-2 sm:px-3 py-1 bg-gray-100 text-gray-700 rounded-md hover:bg-gray-200 transition-colors"
				>
					GitHub
				</a>
				<button
					onclick={() => showSystemMessage = true}
					class="text-xs sm:text-sm px-2 sm:px-3 py-1 bg-purple-100 text-purple-700 rounded-md hover:bg-purple-200 transition-colors"
				>
					System
				</button>
			</div>
		</div>
	</header>


	<!-- System Message Input Modal -->
	{#if showSystemMessage}
		<div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
			<div class="bg-white rounded-lg p-4 sm:p-6 w-full max-w-4xl max-h-[90vh] flex flex-col">
				<h2 class="text-lg font-semibold mb-4">Edit System Message</h2>
				<textarea
					value={getSystemMessage()}
					oninput={(e) => updateSystemMessage((e.target as HTMLTextAreaElement).value)}
					placeholder="Enter system message (e.g., 'You are a helpful AI assistant.')"
					class="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 resize-vertical flex-1 text-sm"
					rows="8"
				></textarea>
				<div class="flex flex-col sm:flex-row gap-2 mt-4 flex-shrink-0">
					<button
						onclick={() => showSystemMessage = false}
						class="flex-1 px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors"
					>
						Save
					</button>
					<button
						onclick={() => showSystemMessage = false}
						class="px-4 py-2 border border-gray-300 text-gray-700 rounded-md hover:bg-gray-50 transition-colors sm:flex-initial"
					>
						Cancel
					</button>
				</div>
			</div>
		</div>
	{/if}

	<!-- Main Content -->
	<main class="flex-1 flex flex-col max-w-4xl mx-auto w-full overflow-hidden">
		<!-- Messages Container -->
		<div bind:this={messagesContainer} class="flex-1 overflow-y-auto px-3 sm:px-4 py-4 pb-20 sm:pb-24">
			{#if messages.filter(m => m.role !== 'system').length === 0}
				<div class="text-center text-gray-500 mt-8">
					<p class="text-lg mb-2">Welcome to markdown-ui-chat!</p>
					<p class="text-sm px-4">Start a conversation by typing a message below.</p>
				</div>
			{/if}

			{#each messages.filter(m => m.role !== 'system') as message (message.id)}
				<div class="mb-4 sm:mb-6 flex {message.role === 'user' ? 'justify-end' : 'justify-start'}">
					<div class="flex w-full max-w-[85%] sm:max-w-xs lg:max-w-2xl {message.role === 'user' ? 'flex-row-reverse' : 'flex-row'} gap-2 sm:gap-3">
						<!-- Avatar -->
						<div class="flex-shrink-0">
							<div class="w-7 h-7 sm:w-8 sm:h-8 rounded-full flex items-center justify-center text-xs sm:text-sm {message.role === 'user' ? 'bg-blue-600 text-white' : 'bg-green-600 text-white'}">
								{message.role === 'user' ? 'U' : 'AI'}
							</div>
						</div>
						
						<!-- Message Bubble -->
						<div class="flex flex-col {message.role === 'user' ? 'items-end' : 'items-start'} flex-1 min-w-0">
							<div class="px-3 sm:px-4 py-2 rounded-lg {message.role === 'user' ? 'bg-blue-600 text-white' : 'bg-white text-gray-900 shadow-sm border border-gray-200'} relative w-full break-words text-sm sm:text-base">
								<MarkdownUI 
                                    html={message.parsedHtml || marked.parse(message.content)} 
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
							<div class="text-xs text-gray-500 mt-1 px-1 sm:px-2">
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
	<div class="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 px-3 sm:px-4 py-3 sm:py-4 z-40 safe-area-inset-bottom">
		<div class="max-w-4xl mx-auto">
			<!-- Suggestion Buttons -->
			{#if showSuggestions}
				<div class="mb-3 flex flex-wrap gap-2 justify-center">
					<button 
						onclick={() => handleSuggestionClick("Who is the cutest cat?")}
						class="px-3 py-2 bg-blue-50 text-blue-700 text-sm rounded-md hover:bg-blue-100 transition-colors border border-blue-200"
					>
						Cutest cat?
					</button>
					<button 
						onclick={() => handleSuggestionClick("Show me coding tips")}
						class="px-3 py-2 bg-green-50 text-green-700 text-sm rounded-md hover:bg-green-100 transition-colors border border-green-200"
					>
						Coding tips
					</button>
					<button 
						onclick={() => handleSuggestionClick("Recommend me some music")}
						class="px-3 py-2 bg-purple-50 text-purple-700 text-sm rounded-md hover:bg-purple-100 transition-colors border border-purple-200"
					>
						Music recommendation
					</button>
				</div>
			{/if}
			
			<div class="flex gap-2 items-end">
				<textarea
					bind:value={inputMessage}
					onkeydown={handleKeyPress}
					placeholder="Type your message... (Enter to send)"
					class="flex-1 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 resize-none text-sm sm:text-base min-h-[40px] max-h-32"
					rows="1"
					disabled={isLoading || isStreaming}
				></textarea>
				<button
					onclick={sendMessage}
					disabled={!inputMessage.trim() || isLoading || isStreaming}
					class="px-3 sm:px-4 py-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 transition-colors disabled:opacity-50 disabled:cursor-not-allowed flex items-center gap-1 sm:gap-2 flex-shrink-0 text-sm sm:text-base min-h-[40px]"
				>
					{#if isLoading}
						<div class="w-3 h-3 sm:w-4 sm:h-4 border-2 border-white border-t-transparent rounded-full animate-spin"></div>
					{:else if isStreaming}
						<div class="flex items-center gap-1">
							<div class="w-1.5 h-1.5 sm:w-2 sm:h-2 bg-white rounded-full animate-bounce"></div>
							<div class="w-1.5 h-1.5 sm:w-2 sm:h-2 bg-white rounded-full animate-bounce" style="animation-delay: 0.1s"></div>
							<div class="w-1.5 h-1.5 sm:w-2 sm:h-2 bg-white rounded-full animate-bounce" style="animation-delay: 0.2s"></div>
						</div>
					{/if}
					<span class="hidden sm:inline">{isStreaming ? 'Streaming...' : 'Send'}</span>
					<span class="sm:hidden">{isStreaming ? '...' : 'Send'}</span>
				</button>
			</div>
		</div>
	</div>
</div>
