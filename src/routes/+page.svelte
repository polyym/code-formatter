<script lang="ts">
	import { onMount } from 'svelte';
	import * as prettier from 'prettier';
	import type { Plugin } from 'prettier';
	import * as babelPlugin from 'prettier/plugins/babel';
	import * as estreePlugin from 'prettier/plugins/estree';
	import * as typescriptPlugin from 'prettier/plugins/typescript';
	import * as Diff from 'diff';
	import Prism from 'prismjs';
	import 'prismjs/components/prism-javascript';
	import 'prismjs/components/prism-typescript';
	import 'prismjs/components/prism-jsx';
	import 'prismjs/components/prism-tsx';
	import 'prismjs/themes/prism-okaidia.css';

	// Constants
	const DEBOUNCE_DELAY_MS = 500;
	const STATUS_DURATION_MS = 2000;
	const ERROR_STATUS_DURATION_MS = 5000;
	const MAX_FILE_SIZE_BYTES = 1024 * 1024;
	const MAX_INPUT_LENGTH = 500000;

	interface LanguageConfig {
		extension: string;
		languageClass: string;
		parser: string;
	}

	const LANGUAGE_CONFIG: Record<string, LanguageConfig> = {
		javascript: { extension: 'js', languageClass: 'language-javascript', parser: 'babel' },
		jsx: { extension: 'jsx', languageClass: 'language-jsx', parser: 'babel' },
		typescript: { extension: 'ts', languageClass: 'language-typescript', parser: 'typescript' },
		tsx: { extension: 'tsx', languageClass: 'language-tsx', parser: 'typescript' }
	};

	const EXTENSION_TO_LANGUAGE: Record<string, string> = {
		js: 'javascript',
		mjs: 'javascript',
		cjs: 'javascript',
		jsx: 'jsx',
		ts: 'typescript',
		tsx: 'tsx'
	};

	const SAMPLE_CODE: Record<string, string> = {
		javascript:
			'function greet(name) {var greeting = "Hello, " + name + "!";[1,2,3].forEach(x=>console.log(x));return greeting;};console.log(greet("World"));',
		jsx: 'import React from "react";function Button({onClick,children,variant="primary",disabled=false}){const baseStyles="px-4 py-2 rounded-lg font-medium transition-all";const variants={primary:"bg-blue-500 hover:bg-blue-600 text-white",secondary:"bg-gray-200 hover:bg-gray-300 text-gray-800"};return(<button onClick={onClick} disabled={disabled} className={`${baseStyles} ${variants[variant]} ${disabled?"opacity-50 cursor-not-allowed":""}`}>{children}</button>);};export default Button;',
		typescript:
			'interface User { name: string; age: number; email?: string; } function processUser(user: User): string { if (!user.name || user.age < 0) { throw new Error("Invalid user data"); } return `User: ${user.name}, Age: ${user.age}${user.email ? ", Email: " + user.email : ""}`; } const users: User[] = [{ name: "Alice", age: 30, email: "alice@example.com" }, { name: "Bob", age: 25 }]; users.forEach(user => console.log(processUser(user)));',
		tsx: 'import React, { useState, useEffect } from "react"; interface Props { title: string; items: string[]; onItemClick?: (item: string) => void; } const ItemList: React.FC<Props> = ({ title, items, onItemClick }) => { const [selectedItem, setSelectedItem] = useState<string | null>(null); useEffect(() => { console.log("Items updated:", items); }, [items]); return (<div className="item-list"><h2>{title}</h2><ul>{items.map((item, index) => (<li key={index} onClick={() => { setSelectedItem(item); onItemClick?.(item); }} className={selectedItem === item ? "selected" : ""}>{item}</li>))}</ul></div>); }; export default ItemList;'
	};

	// State
	let inputCode = $state('');
	let outputCode = $state('');
	let selectedLanguage = $state('auto');
	let status = $state('ready');
	let statusClass = $state('text-gray-400');
	let errorMessage = $state('');
	let showError = $state(false);
	let showSettings = $state(false);
	let isDiffView = $state(false);
	let showDropOverlay = $state(false);

	// Options
	let tabWidth = $state(4);
	let printWidth = $state(88);
	let trailingComma = $state('es5');
	let useSemi = $state(true);
	let useSingleQuote = $state(false);
	let showLineNumbers = $state(true);

	// Internal state
	let lastFormattedCode = '';
	let lastInputCode = '';
	let errorLineNumber: number | null = null;
	let uploadedFileName: string | null = null;
	let formatTimeoutId: ReturnType<typeof setTimeout> | undefined;
	let statusTimeoutId: ReturnType<typeof setTimeout> | undefined;
	let outputDiffInfo: Array<{ type: string; lineNum: number | null }> | null = null;
	let fileInputElement: HTMLInputElement;
	let inputTextarea: HTMLTextAreaElement;
	let outputCodeElement: HTMLElement;
	let inputLineNumbersElement = $state<HTMLElement | null>(null);
	let userHasModifiedInput = false;
	let showLanguageChangeWarning = $state(false);
	let pendingLanguageChange: string | null = null;
	let previousLanguage: string = 'auto';

	// Derived values
	let effectiveLanguage = $derived.by(() => {
		if (selectedLanguage === 'auto') {
			return detectLanguage(inputCode);
		}
		return selectedLanguage;
	});

	let languageConfig = $derived(LANGUAGE_CONFIG[effectiveLanguage] || LANGUAGE_CONFIG.javascript);

	let inputFilename = $derived.by(() => {
		const base = uploadedFileName || `source.${languageConfig.extension}`;
		if (selectedLanguage === 'auto') {
			return `${base} (${effectiveLanguage})`;
		}
		return base;
	});

	let outputFilename = $derived(`formatted.${languageConfig.extension}`);

	let inputStats = $derived.by(() => {
		const chars = inputCode.length;
		const lines = inputCode ? inputCode.split('\n').length : 0;
		return `${lines.toLocaleString('en-GB')} lines, ${chars.toLocaleString('en-GB')} chars`;
	});

	let outputStats = $derived.by(() => {
		const chars = outputCode.length;
		const lines = outputCode ? outputCode.split('\n').length : 0;
		return `${lines.toLocaleString('en-GB')} lines, ${chars.toLocaleString('en-GB')} chars`;
	});

	let inputLineCount = $derived(inputCode ? inputCode.split('\n').length : 1);
	let outputLineCount = $derived(outputCode ? outputCode.split('\n').length : 0);

	// Functions
	function detectLanguage(code: string): string {
		const hasTypeAnnotations =
			/:\s*(string|number|boolean|any|void|never|unknown|object)\b/.test(code) ||
			/interface\s+\w+/.test(code) ||
			/type\s+\w+\s*=/.test(code) ||
			/<[A-Z]\w*>/.test(code) ||
			/:\s*\w+\[\]/.test(code) ||
			/as\s+(const|string|number)/.test(code);

		const hasJsx =
			/<[A-Z][a-zA-Z]*[\s/>]/.test(code) ||
			/<\/[A-Z][a-zA-Z]*>/.test(code) ||
			/className=/.test(code) ||
			/onClick=/.test(code) ||
			/<>[\s\S]*<\/>/.test(code) ||
			/React\./.test(code) ||
			/useState|useEffect|useCallback|useMemo/.test(code);

		if (hasTypeAnnotations && hasJsx) return 'tsx';
		if (hasTypeAnnotations) return 'typescript';
		if (hasJsx) return 'jsx';
		return 'javascript';
	}

	function updateStatus(text: string, colourClass: string, duration = STATUS_DURATION_MS) {
		if (statusTimeoutId !== undefined) {
			clearTimeout(statusTimeoutId);
		}

		status = text;
		statusClass = colourClass;

		statusTimeoutId = setTimeout(() => {
			status = 'ready';
			statusClass = 'text-gray-400';
			statusTimeoutId = undefined;
		}, duration);
	}

	function displayError(message: string) {
		errorMessage = message;
		showError = true;
		updateStatus('error', 'text-pink-400', ERROR_STATUS_DURATION_MS);
	}

	function hideError() {
		showError = false;
		errorLineNumber = null;
	}

	function escapeHtml(text: string): string {
		const div = document.createElement('div');
		div.textContent = text;
		return div.innerHTML;
	}

	function extractErrorLineNumber(message: string): number | null {
		const patterns = [/\((\d+):\d+\)/, /line\s+(\d+)/i, />\s*(\d+)\s*\|/];

		for (const pattern of patterns) {
			const match = message.match(pattern);
			if (match) {
				return parseInt(match[1], 10);
			}
		}
		return null;
	}

	function generateDiffView(
		original: string,
		formatted: string
	): { html: string; diffInfo: Array<{ type: string; lineNum: number | null }> } {
		const diff = Diff.diffLines(original, formatted);
		const htmlParts: string[] = [];
		const diffInfo: Array<{ type: string; lineNum: number | null }> = [];
		let lineNum = 1;

		diff.forEach((part) => {
			const lines = part.value.split('\n');
			if (lines[lines.length - 1] === '') {
				lines.pop();
			}

			lines.forEach((line) => {
				const escapedLine = escapeHtml(line) || '&nbsp;';

				if (part.added) {
					htmlParts.push(`<div class="code-line diff-line-added">${escapedLine}</div>`);
					diffInfo.push({ type: 'added', lineNum: lineNum++ });
				} else if (part.removed) {
					htmlParts.push(`<div class="code-line diff-line-removed">${escapedLine}</div>`);
					diffInfo.push({ type: 'removed', lineNum: null });
				} else {
					htmlParts.push(`<div class="code-line">${escapedLine}</div>`);
					diffInfo.push({ type: 'unchanged', lineNum: lineNum++ });
				}
			});
		});

		return { html: htmlParts.join(''), diffInfo };
	}

	async function formatCode() {
		hideError();

		if (inputCode.length > MAX_INPUT_LENGTH) {
			displayError(
				`Input too large. Maximum is ${MAX_INPUT_LENGTH.toLocaleString('en-GB')} characters. Current: ${inputCode.length.toLocaleString('en-GB')}`
			);
			return;
		}

		lastInputCode = inputCode;

		if (!inputCode.trim()) {
			outputCode = '';
			lastFormattedCode = '';
			outputDiffInfo = null;
			return;
		}

		updateStatus('formatting...', 'text-yellow-400', 1000);

		try {
			const plugins: Plugin[] =
				languageConfig.parser === 'typescript'
					? [typescriptPlugin as unknown as Plugin, estreePlugin as unknown as Plugin]
					: [babelPlugin as unknown as Plugin, estreePlugin as unknown as Plugin];

			const formattedCode = await prettier.format(inputCode, {
				parser: languageConfig.parser,
				plugins,
				tabWidth,
				printWidth,
				trailingComma: trailingComma as 'none' | 'es5' | 'all',
				semi: useSemi,
				singleQuote: useSingleQuote,
				arrowParens: 'always'
			});

			const trimmedCode = formattedCode.trim();
			lastFormattedCode = trimmedCode;
			outputCode = trimmedCode;

			if (isDiffView) {
				const { html, diffInfo } = generateDiffView(inputCode, trimmedCode);
				outputDiffInfo = diffInfo;
				if (outputCodeElement) {
					outputCodeElement.innerHTML = html;
				}
			} else {
				outputDiffInfo = null;
				if (outputCodeElement) {
					outputCodeElement.textContent = trimmedCode;
					Prism.highlightElement(outputCodeElement);
				}
			}

			updateStatus('formatted', 'text-violet-400');
		} catch (error) {
			handleFormatError(error as Error);
		}
	}

	function handleFormatError(error: Error) {
		errorLineNumber = extractErrorLineNumber(error.message);

		const errorLines = error.message.split('\n');
		const shortErrorMessage = errorLines[0] || 'Unknown error';

		displayError(shortErrorMessage);

		outputCode =
			'// Formatting failed. Check code for syntax errors.\n' + `// Error: ${error.message}`;

		outputDiffInfo = null;
		if (outputCodeElement) {
			outputCodeElement.textContent = outputCode;
			Prism.highlightElement(outputCodeElement);
		}
	}

	function handleInputChange() {
		userHasModifiedInput = true;

		if (formatTimeoutId !== undefined) {
			clearTimeout(formatTimeoutId);
		}

		formatTimeoutId = setTimeout(() => {
			formatCode();
			formatTimeoutId = undefined;
		}, DEBOUNCE_DELAY_MS);
	}

	function handleFileUpload(file: File) {
		if (file.size > MAX_FILE_SIZE_BYTES) {
			const maxSizeMB = MAX_FILE_SIZE_BYTES / (1024 * 1024);
			displayError(`File too large. Maximum size is ${maxSizeMB}MB.`);
			return;
		}

		const extension = file.name.split('.').pop()?.toLowerCase() || '';
		const language = EXTENSION_TO_LANGUAGE[extension];

		if (!language) {
			displayError(
				`Unsupported file type: .${extension}. Please use .js, .jsx, .ts, or .tsx files.`
			);
			return;
		}

		const reader = new FileReader();

		reader.onload = (event) => {
			const content = event.target?.result as string;

			if (content.length > MAX_INPUT_LENGTH) {
				displayError(
					`File content too large. Maximum is ${MAX_INPUT_LENGTH.toLocaleString('en-GB')} characters.`
				);
				return;
			}

			inputCode = content;
			uploadedFileName = file.name;
			selectedLanguage = language;
			userHasModifiedInput = true;
			formatCode();
			updateStatus('file loaded', 'text-cyan-400');
		};

		reader.onerror = () => {
			displayError('Failed to read file. Please try again.');
		};

		reader.readAsText(file);
	}

	function downloadFormattedCode() {
		if (!outputCode || outputCode.startsWith('// Formatting failed')) {
			displayError('No formatted code to download.');
			return;
		}

		const filename = `formatted.${languageConfig.extension}`;
		const blob = new Blob([outputCode], { type: 'text/plain;charset=utf-8' });
		const url = URL.createObjectURL(blob);

		const link = document.createElement('a');
		link.href = url;
		link.download = filename;
		link.style.display = 'none';

		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);

		URL.revokeObjectURL(url);
		updateStatus('downloaded', 'text-cyan-400');
	}

	async function pasteFromClipboard() {
		if (!navigator.clipboard || !navigator.clipboard.readText) {
			displayError('Clipboard paste not supported by your browser.');
			return;
		}

		try {
			const text = await navigator.clipboard.readText();

			if (text.length > MAX_INPUT_LENGTH) {
				displayError(
					`Content too large. Maximum is ${MAX_INPUT_LENGTH.toLocaleString('en-GB')} characters.`
				);
				return;
			}

			inputCode = text;
			uploadedFileName = null;
			userHasModifiedInput = true;
			formatCode();
			updateStatus('pasted!', 'text-cyan-400');
		} catch {
			displayError('Failed to read from clipboard. Please grant permission or use Ctrl+V.');
		}
	}

	async function copyToClipboard() {
		if (!outputCode || outputCode.startsWith('// Formatting failed')) {
			displayError('No formatted code to copy.');
			return;
		}

		if (navigator.clipboard && navigator.clipboard.writeText) {
			try {
				await navigator.clipboard.writeText(outputCode);
				updateStatus('copied!', 'text-cyan-400');
				return;
			} catch {
				// Clipboard API failed, fall through to fallback
			}
		}

		copyWithFallback(outputCode);
	}

	function copyWithFallback(text: string) {
		const tempTextArea = document.createElement('textarea');
		tempTextArea.value = text;
		tempTextArea.style.position = 'fixed';
		tempTextArea.style.left = '-9999px';
		tempTextArea.style.top = '-9999px';
		tempTextArea.setAttribute('readonly', '');
		tempTextArea.setAttribute('aria-hidden', 'true');

		document.body.appendChild(tempTextArea);

		let success = false;
		try {
			tempTextArea.select();
			tempTextArea.setSelectionRange(0, text.length);
			success = document.execCommand('copy');
		} catch {
			// execCommand copy failed
		}

		document.body.removeChild(tempTextArea);

		if (success) {
			updateStatus('copied!', 'text-cyan-400');
		} else {
			displayError('Copy failed. Please use Ctrl+C.');
		}
	}

	function clearAll() {
		inputCode = '';
		outputCode = '';
		lastFormattedCode = '';
		lastInputCode = '';
		uploadedFileName = null;
		errorLineNumber = null;
		outputDiffInfo = null;
		userHasModifiedInput = false;
		hideError();
		updateStatus('cleared', 'text-gray-500', 1000);
	}

	function handleLanguageChange(newLanguage: string) {
		if (userHasModifiedInput && inputCode.trim() !== '') {
			// newLanguage is already set via bind:value, so previousLanguage holds the old value
			pendingLanguageChange = newLanguage;
			showLanguageChangeWarning = true;
		} else {
			applyLanguageChange(newLanguage);
		}
	}

	function applyLanguageChange(newLanguage: string) {
		selectedLanguage = newLanguage;
		previousLanguage = newLanguage;
		userHasModifiedInput = false;

		if (newLanguage === 'auto') {
			inputCode = '';
			outputCode = '';
			lastFormattedCode = '';
			lastInputCode = '';
		} else {
			inputCode = SAMPLE_CODE[newLanguage] || '';
			formatCode();
		}

		showLanguageChangeWarning = false;
		pendingLanguageChange = null;
	}

	function confirmLanguageChange() {
		if (pendingLanguageChange !== null) {
			previousLanguage = pendingLanguageChange;
			applyLanguageChange(pendingLanguageChange);
		}
	}

	function cancelLanguageChange() {
		// Restore the previous language selection
		selectedLanguage = previousLanguage;
		showLanguageChangeWarning = false;
		pendingLanguageChange = null;
	}

	function toggleDiffView() {
		isDiffView = !isDiffView;

		if (lastFormattedCode) {
			if (isDiffView) {
				const { html, diffInfo } = generateDiffView(lastInputCode, lastFormattedCode);
				outputDiffInfo = diffInfo;
				if (outputCodeElement) {
					outputCodeElement.innerHTML = html;
				}
			} else {
				outputDiffInfo = null;
				if (outputCodeElement) {
					outputCodeElement.textContent = lastFormattedCode;
					Prism.highlightElement(outputCodeElement);
				}
			}
		}
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
		event.stopPropagation();
		showDropOverlay = true;
	}

	function handleDragLeave(event: DragEvent) {
		event.preventDefault();
		event.stopPropagation();

		const target = event.currentTarget as HTMLElement;
		const rect = target.getBoundingClientRect();
		if (
			event.clientX < rect.left ||
			event.clientX >= rect.right ||
			event.clientY < rect.top ||
			event.clientY >= rect.bottom
		) {
			showDropOverlay = false;
		}
	}

	function handleDrop(event: DragEvent) {
		event.preventDefault();
		event.stopPropagation();
		showDropOverlay = false;

		const files = event.dataTransfer?.files;
		if (files && files.length > 0) {
			handleFileUpload(files[0]);
		}
	}

	function handleKeydown(event: KeyboardEvent) {
		const isModifierPressed = event.ctrlKey || event.metaKey;

		if (!isModifierPressed) return;

		switch (event.key.toLowerCase()) {
			case 'enter':
				event.preventDefault();
				formatCode();
				break;
			case 'k':
				event.preventDefault();
				copyToClipboard();
				break;
			case 's':
				event.preventDefault();
				downloadFormattedCode();
				break;
		}
	}

	function syncInputScroll() {
		if (inputLineNumbersElement && inputTextarea) {
			inputLineNumbersElement.scrollTop = inputTextarea.scrollTop;
		}
	}

	function generateInputLineNumbers(): string {
		const lines: string[] = [];
		for (let i = 1; i <= inputLineCount; i++) {
			const isError = errorLineNumber !== null && i === errorLineNumber;
			const className = isError ? 'text-red-400 font-bold' : '';
			lines.push(`<div class="${className}">${i}</div>`);
		}
		return lines.join('');
	}

	function generateOutputLineNumbers(): string {
		if (outputDiffInfo) {
			return outputDiffInfo
				.map((info) => {
					let className = '';
					if (info.type === 'added') className = 'text-green-400';
					else if (info.type === 'removed') className = 'text-red-400';
					return `<div class="${className}">${info.lineNum || ''}</div>`;
				})
				.join('');
		}

		const lines: string[] = [];
		for (let i = 1; i <= outputLineCount; i++) {
			lines.push(`<div>${i}</div>`);
		}
		return lines.join('');
	}

	onMount(() => {
		// Start with empty input when auto-detect is selected
		// Users can paste code or select a specific language to see sample code

		document.addEventListener('keydown', handleKeydown);

		return () => {
			document.removeEventListener('keydown', handleKeydown);
			if (formatTimeoutId !== undefined) {
				clearTimeout(formatTimeoutId);
			}
			if (statusTimeoutId !== undefined) {
				clearTimeout(statusTimeoutId);
			}
		};
	});
</script>

<svelte:head>
	<title>Free Online Code Formatter - JavaScript, TypeScript, JSX, TSX | Prettier</title>
</svelte:head>

<a href="#main-content" class="skip-to-content">Skip to main content</a>

<div class="min-h-screen p-4 sm:p-6 lg:p-8 flex flex-col">
	<!-- Header -->
	<header class="text-center mb-8">
		<h1 class="text-4xl sm:text-5xl font-bold neon-primary glitch" data-text="CODE.FORMAT">
			CODE.FORMAT
		</h1>
		<p class="mt-3 text-sm text-gray-400 terminal-prompt">Multi-Language Formatter • JS/JSX/TS/TSX</p>
	</header>

	<!-- Main Content -->
	<main id="main-content" class="max-w-7xl w-full mx-auto flex-grow flex flex-col">
		<!-- Controls -->
		<div class="mb-6 flex flex-col gap-4 p-4 bg-black/30 border border-gray-800 rounded-2xl">
			<!-- Top row -->
			<div class="flex flex-col sm:flex-row justify-between items-center gap-4">
				<div class="flex items-center gap-4 flex-wrap">
					<label class="text-sm text-gray-300" for="language-select">Language:</label>
					<select
						id="language-select"
						bind:value={selectedLanguage}
						onchange={() => handleLanguageChange(selectedLanguage)}
						class="bg-gray-900 border border-gray-700 rounded-lg px-3 py-2 text-sm focus:border-indigo-500 focus:outline-none focus:ring-1 focus:ring-indigo-500 text-white"
						aria-label="Select programming language"
					>
						<option value="auto">Auto-detect</option>
						<option value="javascript">JavaScript</option>
						<option value="jsx">JSX</option>
						<option value="typescript">TypeScript</option>
						<option value="tsx">TSX</option>
					</select>
					<div id="status-container" class="text-xs text-gray-500 pl-4" role="status" aria-live="polite">
						Status: <span class="font-semibold {statusClass}">{status}</span>
					</div>
				</div>

				<div class="flex gap-3 flex-wrap">
					<button
						type="button"
						class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider"
						class:active={showSettings}
						aria-label="Toggle formatting options"
						aria-expanded={showSettings}
						onclick={() => (showSettings = !showSettings)}
					>
						[Options]
					</button>
					<button
						type="button"
						class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider"
						class:active={isDiffView}
						aria-label="Toggle diff view"
						aria-pressed={isDiffView}
						onclick={toggleDiffView}
					>
						[Diff]
					</button>
				</div>
			</div>

			<!-- Settings Panel (collapsible) -->
			<div class="settings-panel" class:open={showSettings}>
				<div class="pt-4 border-t border-gray-800">
					<div class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-6 gap-4">
						<!-- Tab Width -->
						<div class="flex flex-col gap-1">
							<label class="text-xs text-gray-400" for="opt-tab-width">Tab Width</label>
							<select
								id="opt-tab-width"
								bind:value={tabWidth}
								onchange={() => formatCode()}
								class="bg-gray-900 border border-gray-700 rounded px-2 py-1.5 text-sm text-white"
							>
								<option value={2}>2</option>
								<option value={4}>4</option>
								<option value={8}>8</option>
							</select>
						</div>

						<!-- Print Width -->
						<div class="flex flex-col gap-1">
							<label class="text-xs text-gray-400" for="opt-print-width">Print Width</label>
							<select
								id="opt-print-width"
								bind:value={printWidth}
								onchange={() => formatCode()}
								class="bg-gray-900 border border-gray-700 rounded px-2 py-1.5 text-sm text-white"
							>
								<option value={80}>80</option>
								<option value={88}>88</option>
								<option value={100}>100</option>
								<option value={120}>120</option>
							</select>
						</div>

						<!-- Trailing Comma -->
						<div class="flex flex-col gap-1">
							<label class="text-xs text-gray-400" for="opt-trailing-comma">Trailing Comma</label>
							<select
								id="opt-trailing-comma"
								bind:value={trailingComma}
								onchange={() => formatCode()}
								class="bg-gray-900 border border-gray-700 rounded px-2 py-1.5 text-sm text-white"
							>
								<option value="none">None</option>
								<option value="es5">ES5</option>
								<option value="all">All</option>
							</select>
						</div>

						<!-- Semicolons -->
						<div class="flex flex-col gap-1">
							<span class="text-xs text-gray-400">Semicolons</span>
							<div class="toggle-container h-[30px] items-center">
								<button
									type="button"
									class="toggle-switch"
									class:active={useSemi}
									role="switch"
									aria-checked={useSemi}
									aria-label="Use semicolons"
									onclick={() => {
										useSemi = !useSemi;
										formatCode();
									}}
								></button>
								<span class="text-xs text-gray-500">{useSemi ? 'On' : 'Off'}</span>
							</div>
						</div>

						<!-- Single Quotes -->
						<div class="flex flex-col gap-1">
							<span class="text-xs text-gray-400">Single Quotes</span>
							<div class="toggle-container h-[30px] items-center">
								<button
									type="button"
									class="toggle-switch"
									class:active={useSingleQuote}
									role="switch"
									aria-checked={useSingleQuote}
									aria-label="Use single quotes"
									onclick={() => {
										useSingleQuote = !useSingleQuote;
										formatCode();
									}}
								></button>
								<span class="text-xs text-gray-500">{useSingleQuote ? 'On' : 'Off'}</span>
							</div>
						</div>

						<!-- Line Numbers -->
						<div class="flex flex-col gap-1">
							<span class="text-xs text-gray-400">Line Numbers</span>
							<div class="toggle-container h-[30px] items-center">
								<button
									type="button"
									class="toggle-switch"
									class:active={showLineNumbers}
									role="switch"
									aria-checked={showLineNumbers}
									aria-label="Show line numbers"
									onclick={() => (showLineNumbers = !showLineNumbers)}
								></button>
								<span class="text-xs text-gray-500">{showLineNumbers ? 'On' : 'Off'}</span>
							</div>
						</div>
					</div>
				</div>
			</div>

			<!-- Action buttons -->
			<div class="flex flex-wrap gap-3">
				<input
					type="file"
					bind:this={fileInputElement}
					class="file-input"
					accept=".js,.jsx,.ts,.tsx,.mjs,.cjs"
					aria-label="Upload code file"
					onchange={(e) => {
						const target = e.target as HTMLInputElement;
						const files = target.files;
						if (files && files.length > 0) {
							handleFileUpload(files[0]);
						}
						target.value = '';
					}}
				/>
				<button
					type="button"
					class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider"
					aria-label="Upload code file"
					onclick={() => fileInputElement?.click()}
				>
					[Upload]
				</button>
				<button
					type="button"
					class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider"
					aria-label="Download formatted code"
					onclick={downloadFormattedCode}
				>
					[Download]
				</button>
				<button
					type="button"
					class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider"
					aria-label="Paste code from clipboard"
					onclick={pasteFromClipboard}
				>
					[Paste]
				</button>
				<button
					type="button"
					class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider"
					aria-label="Copy formatted code to clipboard"
					onclick={copyToClipboard}
				>
					[Copy]
				</button>
				<button
					type="button"
					class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider text-pink-500 hover:text-pink-400"
					aria-label="Clear all code"
					onclick={clearAll}
				>
					[Clear]
				</button>
				<div class="flex-grow"></div>
				<button
					type="button"
					class="btn-cyber btn-primary px-4 py-2 text-sm rounded-lg uppercase tracking-wider font-bold"
					aria-label="Format code"
					onclick={() => formatCode()}
				>
					[Format]
				</button>
			</div>
		</div>

		<!-- Error Display -->
		{#if showError}
			<div class="mb-4" role="alert" aria-live="assertive">
				<div class="bg-pink-900/30 border border-pink-500/50 rounded-xl p-3 text-pink-400 text-sm">
					<span class="font-bold">[ERROR]</span>
					<span>{errorMessage}</span>
				</div>
			</div>
		{/if}

		<!-- Language Change Warning Modal -->
		{#if showLanguageChangeWarning}
			<div class="fixed inset-0 bg-black/70 flex items-center justify-center z-50" role="dialog" aria-modal="true" aria-labelledby="warning-title">
				<div class="bg-gray-900 border border-yellow-500/50 rounded-2xl p-6 max-w-md mx-4 shadow-2xl">
					<h2 id="warning-title" class="text-lg font-bold text-yellow-400 mb-3">[WARNING] Reset Input?</h2>
					<p class="text-gray-300 text-sm mb-6">
						Changing the language will replace your current code with sample code for the selected language. Your changes will be lost.
					</p>
					<div class="flex gap-3 justify-end">
						<button
							type="button"
							class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider"
							onclick={cancelLanguageChange}
						>
							[Cancel]
						</button>
						<button
							type="button"
							class="btn-cyber px-4 py-2 text-sm rounded-lg uppercase tracking-wider text-yellow-400 hover:text-yellow-300 border-yellow-500/50"
							onclick={confirmLanguageChange}
						>
							[Continue]
						</button>
					</div>
				</div>
			</div>
		{/if}

		<!-- Code Areas -->
		<div class="flex-grow grid grid-cols-1 xl:grid-cols-2 gap-6">
			<!-- Input -->
			<div
				class="terminal-area rounded-2xl flex flex-col relative"
				ondragover={handleDragOver}
				ondragleave={handleDragLeave}
				ondrop={handleDrop}
				role="region"
				aria-label="Code input area"
			>
				<div class="drop-overlay" class:active={showDropOverlay}>Drop file here to upload</div>
				<div
					class="border-b border-gray-800 p-3 text-xs text-cyan-400 uppercase tracking-wider flex justify-between"
				>
					<span>[Input] {inputFilename}</span>
					<span class="text-gray-500 normal-case">{inputStats}</span>
				</div>
				<div class="input-wrapper">
					{#if showLineNumbers}
						<div
							class="input-line-numbers"
							bind:this={inputLineNumbersElement}
							aria-hidden="true"
						>
							{@html generateInputLineNumbers()}
						</div>
					{/if}
					<textarea
						bind:this={inputTextarea}
						bind:value={inputCode}
						oninput={handleInputChange}
						onscroll={syncInputScroll}
						class="w-full flex-grow bg-transparent text-white caret-indigo-400 focus:outline-none text-sm p-4"
						style="line-height: 1.5rem; resize: none;"
						placeholder="// Paste your code here (Ctrl+V / Cmd+V) or drag and drop a file..."
						spellcheck="false"
						aria-label="Code input"
					></textarea>
				</div>
			</div>

			<!-- Output -->
			<div class="terminal-area rounded-2xl flex flex-col" role="region" aria-label="Formatted code output area">
				<div
					class="border-b border-gray-800 p-3 text-xs text-violet-400 uppercase tracking-wider flex justify-between"
				>
					<span>[Output] {outputFilename}</span>
					<span class="text-gray-500 normal-case">{outputStats}</span>
				</div>
				<div class="flex-grow overflow-auto">
					<div class="line-numbers-container">
						{#if showLineNumbers && outputLineCount > 0}
							<div class="line-numbers" aria-hidden="true">
								{@html generateOutputLineNumbers()}
							</div>
						{/if}
						<div class="code-content">
							<pre><code bind:this={outputCodeElement} class={languageConfig.languageClass} aria-label="Formatted code output">{outputCode}</code></pre>
						</div>
					</div>
				</div>
			</div>
		</div>
	</main>

	<!-- Footer -->
	<footer class="mt-8 text-center text-xs text-gray-600">
		<div class="terminal-prompt">
			Shortcuts: [Ctrl+V] Paste | [Ctrl+K] Copy | [Ctrl+S] Download | [Ctrl+Enter] Format
		</div>
		<div class="mt-4 text-gray-500">
			Created by <a
				href="https://github.com/polyym"
				target="_blank"
				rel="noopener noreferrer"
				class="text-indigo-400 hover:text-indigo-300 transition-colors">polyym</a
			>
			·
			<a
				href="https://github.com/polyym/code-formatter"
				target="_blank"
				rel="noopener noreferrer"
				class="text-indigo-400 hover:text-indigo-300 transition-colors">View Source</a
			>
			· <span class="text-gray-600">v1.0.0</span>
		</div>
	</footer>
</div>
