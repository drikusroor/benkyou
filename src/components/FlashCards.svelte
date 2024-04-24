<script lang="ts">
	import { onMount } from 'svelte';
	import { goto } from '$app/navigation';
	import { base } from '$app/paths';

    export let flashCardsConfigurationData: FlashCardsConfigurationData;

	type TranslationKeys =
		| 'title'
		| 'mostImportantWords'
		| 'answer'
		| 'submitAnswerButton'
		| 'correct'
		| 'incorrect'
		| 'correctAnswers'
		| 'nextQuestion'
		| 'search'
		| 'fromLinkText'
		| 'toLinkText'
		| 'showPronunciationText'
		| 'hidePronunciationText';

	interface Language {
		code: string;
		label: string;
		default?: boolean;
	}

	interface FlashCardsConfigurationData {
		languages: Language[];
		translations: {
			[language: string]: { [key in TranslationKeys]: string };
		};
		questions: Sentence[];
	}

	const data: FlashCardsConfigurationData = flashCardsConfigurationData;

	type LanguageCode = (typeof languages)[number]['code'];

	interface Sentence {
		q: string;
		[key: LanguageCode]: string[];
	}

	type Mistakes = { [question: string]: number };

	let loading = true;
	let languages = data.languages;
	const defaultLanguage = languages.find((language) => language.default);

	let sentences: FlashCardsConfigurationData['questions'];
	let translations: FlashCardsConfigurationData['translations'] = data.translations;
	let t: (key: TranslationKeys, defaultValue?: string) => string;
	let currentLanguage: LanguageCode;
	let learningMode: 'from' | 'to';
	let showPronunciation: boolean = true;

	// QA flow related
	let questionBoxClasses = '';
	let questionText = '';
	let answerInputValue = '';
	let currentSentence: Sentence;
	let mistakes: Mistakes = {};
	let isFeedbackDisplayed = false;
	let currentTimeout: NodeJS.Timeout;
	let submitAnswerButtonText = '';
	let feedbackText = '';

	// Cheatsheet / search related
	let filteredSentences: FlashCardsConfigurationData['questions'] = [];
	let searchInputValue = '';

	// Streak related
	let streakText = 0;
	let streakClasses = 'hidden';

	function handleAnswerSubmit(e: Event) {
		e.preventDefault();

		if (isFeedbackDisplayed) {
			loadNextSentence();
		} else {
			checkAnswer();
		}
	}

	function getQuestionText(currentSentence: Sentence, { learningMode = 'from', showPronunciation = true }) {
		let questionText = currentSentence.q;

		if (learningMode === 'to') {
			questionText = currentSentence[currentLanguage].join(' / ');
		}

		if (!showPronunciation) {
			// Remove the pronunciation between the parentheses from the question text
			const re = /\(.*?\)/;
			return questionText.replace(re, '').trim();
		}

		return questionText;
	}

	/**
	 * Reloads the current sentence with the current settings.
	 * This is useful when the user changes the learning mode or the show pronunciation setting.
	 */
	function reloadCurrentSentence() {
		questionText = getQuestionText(currentSentence, { learningMode, showPronunciation });
	}

	function loadNextSentence() {
		currentTimeout ? clearTimeout(currentTimeout) : null;
		currentSentence = getNextSentence();
		questionText = getQuestionText(currentSentence, { learningMode, showPronunciation });
		answerInputValue = '';
		feedbackText = '';
		isFeedbackDisplayed = false;
		submitAnswerButtonText = getTranslation('submitAnswerButton', 'Submit');
		questionBoxClasses = 'border-purple-600';
	}

	function getNextSentence() {
		let prioritizedSentences = shuffle(sentences)
			.filter((sentence) => sentence.q !== currentSentence?.q)
			.sort((a, b) => {
				if (mistakes[a.q] && mistakes[b.q]) {
					return mistakes[a.q] - mistakes[b.q];
				}

				if (mistakes[a.q]) {
					return -1;
				}

				if (mistakes[b.q]) {
					return 1;
				}

				return 0;
			})
			.slice(0, 10);

		if (prioritizedSentences.length) {
			return prioritizedSentences[Math.floor(Math.random() * prioritizedSentences.length)];
		}

		return sentences[Math.floor(Math.random() * sentences.length)];
	}

	function getTranslation(key: TranslationKeys, defaultValue?: string) {
		const translation = translations[currentLanguage][key] || translations['general'][key];

		if (translation) {
			return translation;
		}

		if (defaultValue) {
			return defaultValue;
		}

		return `{{${key}}}`;
	}

	$: t = (key: TranslationKeys, defaultValue?: string) => {
		if (translations && translations[currentLanguage]) {
			return getTranslation(key, defaultValue);
		}

		return defaultValue || `{{${key}}}`;
	};

	function shuffle<T>(array: T[] = []): T[] {
		let currentIndex = array.length,
			randomIndex;

		// While there remain elements to shuffle.
		while (currentIndex > 0) {
			// Pick a remaining element.
			randomIndex = Math.floor(Math.random() * currentIndex);
			currentIndex--;

			// And swap it with the current element.
			[array[currentIndex], array[randomIndex]] = [array[randomIndex], array[currentIndex]];
		}

		return array;
	}

	function levenshtein(a: string, b: string) {
		const matrix = [];

		if (a.length == 0) return b.length;
		if (b.length == 0) return a.length;

		for (let i = 0; i <= b.length; i++) {
			matrix[i] = [i];
		}

		for (let j = 0; j <= a.length; j++) {
			matrix[0][j] = j;
		}

		for (let i = 1; i <= b.length; i++) {
			for (let j = 1; j <= a.length; j++) {
				if (b.charAt(i - 1) == a.charAt(j - 1)) {
					matrix[i][j] = matrix[i - 1][j - 1];
				} else {
					matrix[i][j] = Math.min(
						matrix[i - 1][j - 1] + 1,
						Math.min(matrix[i][j - 1] + 1, matrix[i - 1][j] + 1)
					);
				}
			}
		}

		return matrix[b.length][a.length];
	}

	function updateStreak(increment = false) {
		if (increment) {
			streakText = streakText + 1;
			streakClasses = 'bg-green-500';
		} else {
			streakText = 0;
			streakClasses = 'bg-pink-500';
		}
	}

	function checkAnswer() {
		const userAnswer = answerInputValue.trim();
		const correctAudio = document.getElementById('correctAudio') as HTMLAudioElement;
		const incorrectAudio = document.getElementById('incorrectAudio') as HTMLAudioElement;
		let isCorrect = false;

		if (!userAnswer) {
			return;
		}

		if (learningMode === 'from') {
			isCorrect = currentSentence[currentLanguage].some(
				(translation) => levenshtein(userAnswer.toLowerCase(), translation.toLowerCase()) <= 2
			);
		} else {
			// Check if the user has entered the exact answer (e.g. "百" instead of "hyaku")
			const exactAnswerMatches = currentSentence.q.match(/^(.*?)\(/);
			const exactAnswer = exactAnswerMatches
				? exactAnswerMatches[1].replace(/[\s,]/g, '')
				: currentSentence.q.replace(/[\s,]/g, '');
			const exactIsCorrect = levenshtein(userAnswer.toLowerCase(), exactAnswer.toLowerCase()) <= 2;

			// Check if the user has entered the localized answer (e.g. "hyaku" instead of "百")
			const localizedAnswerMatches = currentSentence.q.match(/\((.*?)\)/);
			const localizedAnswer = localizedAnswerMatches ? localizedAnswerMatches[1] : '';
			const localizedIsCorrect =
				levenshtein(userAnswer.toLowerCase(), localizedAnswer.toLowerCase()) <= 2;

			isCorrect = localizedIsCorrect || exactIsCorrect;
		}

		const correctAnswers =
			learningMode === 'from' ? currentSentence[currentLanguage].join(', ') : currentSentence.q;

		if (isCorrect) {
			const correctTranslation = getTranslation('correct', 'Correct!');
			feedbackText = `✅ ${correctTranslation} ${correctAnswers}`;
			if (mistakes[currentSentence.q]) {
				mistakes[currentSentence.q]--;
			}
			isFeedbackDisplayed = true;
			submitAnswerButtonText = getTranslation('nextQuestion', 'Next question');
			correctAudio.play();
			questionBoxClasses = 'border-green-500';
			updateStreak(true);
		} else {
			const incorrectTranslation = getTranslation('incorrect', 'Incorrect! Correct answers: ');
			feedbackText = `❌ ${incorrectTranslation} ${correctAnswers}`;
			if (!mistakes[currentSentence.q]) {
				mistakes[currentSentence.q] = 1;
			} else {
				mistakes[currentSentence.q]++;
			}
			isFeedbackDisplayed = true;
			submitAnswerButtonText = getTranslation('nextQuestion', 'Next question');
			incorrectAudio.play();
			questionBoxClasses = 'border-red-500';
			updateStreak(false);
		}

		localStorage.setItem('mistakes', JSON.stringify(mistakes));

		currentTimeout ? clearTimeout(currentTimeout) : null;

		currentTimeout = setTimeout(loadNextSentence, 3500);
	}

	function handleSearchInput() {
		const searchTerm = searchInputValue;

		filteredSentences = sentences.filter((sentence) => {
			return (
				sentence.q.toLowerCase().includes(searchTerm) ||
				sentence[currentLanguage].join(' / ').toLowerCase().includes(searchTerm)
			);
		});
	}

	function changeLanguage(event: Event) {
		event.preventDefault();
		const target = event.currentTarget as HTMLLinkElement;
		const url = new URL(target.href);
		goto(url.href, { replaceState: true });
		const newLanguage = url.searchParams.get('lang') as LanguageCode;
		currentLanguage = newLanguage;
		loadNextSentence();
	}

	function changeMode(event: Event) {
		event.preventDefault();
		const target = event.currentTarget as HTMLLinkElement;
		const url = new URL(target.href);
		goto(url.href, { replaceState: true });
		const newMode = url.searchParams.get('mode') as 'from' | 'to';
		learningMode = newMode;
		loadNextSentence();
	}

	function changeShowPronunciation(event: Event) {
		event.preventDefault();
		const target = event.currentTarget as HTMLLinkElement;
		const url = new URL(target.href);
		goto(url.href, { replaceState: true });
		const newShowPronunciation = url.searchParams.get('showPronunciation') as 'true' | 'false';
		showPronunciation = newShowPronunciation === 'true';
		reloadCurrentSentence();
	}

	onMount(() => {
		currentLanguage = getCurrentLanguage();
		learningMode = getMode();
		showPronunciation = getShowPronunciation();
		sentences = shuffle(data.questions);
		filteredSentences = [...sentences];

		function getCurrentLanguage() {
			const lang = new URLSearchParams(window.location.search).get('lang');

			const exists = languages.some((language) => language.code === lang);

			if (exists) {
				return lang as LanguageCode;
			}

			if (defaultLanguage) {
				return defaultLanguage.code;
			}

			return languages[0].code;
		}

		function getMode() {
			return new URLSearchParams(window.location.search).get('mode') === 'to' ? 'to' : 'from';
		}

		function getShowPronunciation() {
			return new URLSearchParams(window.location.search).get('showPronunciation') !== 'false';
		}

		if (localStorage.getItem('mistakes')) {
			mistakes = JSON.parse(localStorage.getItem('mistakes') || '{}') as Mistakes;
		}

		loadNextSentence();

		loading = false;
	});
</script>

<div class="min-h-screen bg-black flex flex-col items-center p-5">
	<a
		href="{base}/"
		class="sm:absolute mb-5 top-5 left-5 text-white text-sm bg-purple-600 px-4 py-2 rounded-md hover:bg-pink-600 focus:outline-none focus:border-pink-700 focus:ring focus:ring-purple-400 active:bg-pink-700 transition duration-150 ease-in-out"
		>Back to Home</a
	>
	<div class="flex flex-row gap-4">
		{#if !loading}
			<div class="rounded-xl bg-slate-900 p-4 mb-4 flex flex-row gap-2">
				{#each languages as language, index}
					<a
						href="{base}?lang={language.code}&mode={learningMode}&showPronunciation={showPronunciation}"
						class="text-white hover:text-pink-400 {language.code === currentLanguage
							? 'underline'
							: ''}"
						on:click={changeLanguage}
					>
						{language.label}
					</a>
					{#if index < languages.length - 1}
						<span class="text-white">|</span>
					{/if}
				{/each}
			</div>

			<div class="rounded-xl bg-slate-900 p-4 mb-4 flex flex-row gap-2">
				<a
					id="fromLink"
					href="{base}?lang={currentLanguage}&mode=from&showPronunciation={showPronunciation}"
					class="text-white hover:text-pink-400 {learningMode === 'from' ? 'underline' : ''}"
					on:click={changeMode}
				>
					{t('fromLinkText', 'From')}
				</a>
				<span class="text-white">|</span>
				<a
					id="toLink"
					href="{base}?lang={currentLanguage}&mode=to&showPronunciation={showPronunciation}"
					class="text-white hover:text-pink-400 {learningMode === 'to' ? 'underline' : ''}"
					on:click={changeMode}
				>
					{t('toLinkText', 'To')}
				</a>
			</div>

			<div class="rounded-xl bg-slate-900 p-4 mb-4 flex flex-row gap-2">
				<a
					id="showPronunciation"
					href="{base}?lang={currentLanguage}&mode={learningMode}&showPronunciation=true"
					class="text-white hover:text-pink-400 {showPronunciation ? 'underline' : ''}"
					on:click={changeShowPronunciation}
				>
					{t('showPronunciationText', 'は (ha)')}
				</a>
				<span class="text-white">|</span>
				<a
					id="hidePronunciation"
					href="{base}?lang={currentLanguage}&mode={learningMode}&showPronunciation=false"
					class="text-white hover:text-pink-400 {!showPronunciation ? 'underline' : ''}"
					on:click={changeShowPronunciation}
				>
					{t('hidePronunciationText', 'は')}
				</a>
			</div>

		{/if}

		{#if loading}
			<div class="rounded-xl bg-purple-900 animate-pulse p-4 mb-4 flex flex-row gap-2">
				<span class="text-white w-12 text-center">...</span>
				<span class="text-white">|</span>
				<span class="text-white w-12 text-center">...</span>
			</div>

			<div class="rounded-xl bg-purple-900 animate-pulse p-4 mb-4 flex flex-row gap-2">
				<span class="text-white w-12 text-center">...</span>
				<span class="text-white">|</span>
				<span class="text-white w-12 text-center">...</span>
			</div>

			<div class="rounded-xl bg-purple-900 animate-pulse p-4 mb-4 flex flex-row gap-2">
				<span class="text-white w-12 text-center">...</span>
				<span class="text-white">|</span>
				<span class="text-white w-12 text-center">...</span>
			</div>
		{/if}
	</div>

	{#if !loading}
		<div
			class="bg-opacity-60 shadow-lg rounded-xl p-8 w-full max-w-xl border bg-slate-900 transition-colors relative {questionBoxClasses}"
			id="questionBox"
		>
			<div
				class="flex items-center justify-center absolute -top-3 -right-3 md:-top-6 md:-right-6 w-12 h-12 rounded-full text-white px-4 py-2 text-sm font-medium tracking-wider {streakClasses}"
			>
				<span id="streak">
					{streakText || 0}
				</span>
			</div>
			<h1 id="title" class="text-4xl font-bold mb-2 text-pink-400">百 - Hyaku</h1>
			<h2 id="subTitle" class="text-md font-semibold mb-6 text-purple-500 tracking-wider">
				{t('mostImportantWords')}
			</h2>

			<form id="answerForm" class="space-y-4" on:submit={handleAnswerSubmit}>
				<div id="question" class="text-xl text-white font-medium">
					{questionText}
				</div>

				<div>
					<input
						type="text"
						id="answer"
						placeholder={t('answer')}
						required
						class="w-full p-3 border border-purple-400 bg-opacity-50 bg-slate-800 text-white rounded-md focus:outline-none focus:border-pink-400 focus:ring focus:ring-purple-300 transition duration-150 ease-in-out"
						bind:value={answerInputValue}
					/>
				</div>

				<div>
					<button
						type="submit"
						id="submitAnswerButton"
						class="bg-purple-600 text-white px-6 py-3 rounded-md hover:bg-pink-600 focus:outline-none focus:border-pink-700 focus:ring focus:ring-purple-400 active:bg-pink-700 transition duration-150 ease-in-out"
					>
						{t('submitAnswerButton')}
					</button>
				</div>

				<div id="feedback" class="text-lg text-pink-400 font-medium">
					{feedbackText}
				</div>
			</form>
		</div>
	{/if}

	{#if loading}
		<div
			class="bg-opacity-60 shadow-lg rounded-xl p-8 w-full max-w-xl border bg-purple-900 animate-pulse transition-colors relative"
		>
			<div
				class="flex items-center justify-center absolute -top-3 -right-3 md:-top-6 md:-right-6 w-12 h-12 rounded-full text-white px-4 py-2 text-sm font-medium tracking-wider bg-pink-500"
			/>
			<h1 class="text-4xl font-bold mb-2 text-pink-400">...</h1>
			<h2 class="text-md font-semibold mb-6 text-purple-500 tracking-wider">...</h2>

			<div class="space-y-4">
				<div class="text-xl text-white font-medium">...</div>

				<div>
					<input
						type="text"
						id="answer"
						placeholder="..."
						required
						class="w-full p-3 border border-purple-400 bg-opacity-50 bg-slate-800 text-white rounded-md focus:outline-none focus:border-pink-400 focus:ring focus:ring-purple-300 transition duration-150 ease-in-out"
					/>
				</div>

				<div>
					<div class="bg-purple-600 text-white px-6 py-3 rounded-md">...</div>
				</div>
			</div>
		</div>
	{/if}

	{#if !loading}
		<div class="mt-8 bg-slate-900">
			<form class="flex items-center justify-center space-x-4 p-4">
				<input
					type="text"
					id="searchCheatsheet"
					placeholder={t('search')}
					class="w-full p-3 border border-purple-400 bg-opacity-50 bg-slate-800 text-white rounded-md focus:outline-none focus:border-pink-400 focus:ring focus:ring-purple-300 transition duration-150 ease-in-out"
					bind:value={searchInputValue}
					on:input={handleSearchInput}
				/>
			</form>

			<ul class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 p-4" id="cheatsheet">
				{#each filteredSentences as sentence}
					<li
						class="bg-slate-800 p-4 rounded-md shadow-md text-white text-lg font-medium flex items-center justify-between"
					>
						<div class="text-xl">{sentence.q}</div>
						<div class="text-pink-400">{sentence[currentLanguage].join(' / ')}</div>
					</li>
				{/each}
			</ul>
		</div>
	{/if}

	{#if loading}
		<div class="mt-8 bg-purple-900 animate-pulse w-full">
			<div class="flex items-center justify-center space-x-4 p-4">
				<div
					class="w-full p-3 border border-purple-400 bg-opacity-50 bg-slate-800 text-white rounded-md focus:outline-none focus:border-pink-400 focus:ring focus:ring-purple-300 transition duration-150 ease-in-out"
				>
					...
				</div>
			</div>

			<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4 p-4">
				{#each Array(12).fill(true) as number}
					<div
						class="bg-slate-800 p-4 rounded-md shadow-md text-white text-lg font-medium flex items-center justify-between"
					>
						<div class="text-xl">...</div>
						<div class="text-pink-400">...</div>
					</div>
				{/each}
			</div>
		</div>
	{/if}
</div>

<audio id="correctAudio" src="{base}/correct.m4a" />
<audio id="incorrectAudio" src="{base}/incorrect.m4a" />
