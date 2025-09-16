<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Music Match</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/tone@14.7.58/build/Tone.js"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #000;
            color: #fff;
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            padding: 1rem;
            overflow-x: hidden;
        }

        .note {
            width: 3.5rem;
            height: 3.5rem;
            display: flex;
            justify-content: center;
            align-items: center;
            border-radius: 9999px;
            font-size: 1.5rem;
            transition: all 0.2s ease-in-out;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1), 0 1px 3px rgba(0, 0, 0, 0.08);
            user-select: none;
            -webkit-user-select: none;
            -moz-user-select: none;
        }
        .note:hover {
            transform: scale(1.05);
        }
        .note:active {
            transform: scale(0.95);
            box-shadow: inset 0 2px 4px rgba(0, 0, 0, 0.1);
        }

        .note-white {
            background-color: #f3f4f6;
            color: #1f2937;
        }

        .note-black {
            background-color: #1f2937;
            color: #f3f4f6;
        }

        .note-label {
            font-weight: bold;
            font-size: 0.8rem;
            margin-top: 0.5rem;
            text-align: center;
        }

        .note-container {
            display: flex;
            flex-direction: column;
            align-items: center;
            margin: 0.25rem;
        }

        .game-row {
            display: flex;
            justify-content: center;
            width: 100%;
            max-width: 60rem;
            margin-bottom: 2rem;
            flex-wrap: wrap;
        }

        .note-emoji {
            font-size: 2rem;
            cursor: pointer;
            user-select: none;
            transition: all 0.2s ease-in-out;
            transform: scale(1);
        }

        .note-emoji.selected {
            outline: 2px solid #3b82f6;
            outline-offset: 4px;
            box-shadow: 0 0 10px #3b82f6;
        }

        .correct-note {
            background-color: #16a34a !important; /* Tailwind's bg-green-600 */
            color: white;
            cursor: default;
        }

        .incorrect-match {
            animation: shake 0.5s ease-out;
        }

        @keyframes pulse-correct {
            0% { transform: scale(1); box-shadow: 0 0 0 0 rgba(34, 197, 94, 0.7); }
            70% { transform: scale(1.1); box-shadow: 0 0 0 10px rgba(34, 197, 94, 0); }
            100% { transform: scale(1); box-shadow: 0 0 0 0 rgba(34, 197, 94, 0); }
        }

        @keyframes shake {
            0% { transform: translateX(0); }
            20% { transform: translateX(-5px); }
            40% { transform: translateX(5px); }
            60% { transform: translateX(-5px); }
            80% { transform: translateX(5px); }
            100% { transform: translateX(0); }
        }
    </style>
</head>
<body class="bg-black text-gray-100 flex flex-col items-center justify-center p-4">

    <!-- Header -->
    <div class="text-center mb-8">
        <h1 class="text-4xl md:text-5xl font-bold mb-2 text-white drop-shadow-md">Music Match</h1>
        <p class="text-lg text-gray-400">Listen to a note in the top row, then find and tap its match below.</p>
    </div>

    <div id="game-container" class="w-full max-w-5xl rounded-lg p-6 flex flex-col items-center">

        <!-- Row 1: Note Labels -->
        <div id="note-labels" class="game-row"></div>

        <!-- Row 2: Reference Notes with Emojis (Clickable) -->
        <div id="reference-notes" class="game-row"></div>

        <!-- Row 4: Random Music Emojis (Tap to select) -->
        <div id="draggable-emojis" class="game-row p-4 bg-gray-900 rounded-lg shadow-inner"></div>

        <!-- Feedback & Controls -->
        <div class="mt-8 text-center flex flex-col md:flex-row items-center gap-4">
            <button id="confirm-button" class="bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-6 rounded-full shadow-lg transition-all duration-200" disabled>Confirm Match</button>
            <div id="feedback-message" class="text-xl font-bold rounded-full py-2 px-6 transition-all duration-300"></div>
            <button id="reset-button" class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-6 rounded-full shadow-lg transition-all duration-200">Reset</button>
        </div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            const noteOrder = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B', 'C2'];
            const noteColors = ['white', 'black', 'white', 'black', 'white', 'white', 'black', 'white', 'black', 'white', 'black', 'white', 'white'];
            const singleEmoji = '🎵';
            const noteFrequencies = {
                'C': 'C4', 'C#': 'C#4', 'D': 'D4', 'D#': 'D#4', 'E': 'E4', 'F': 'F4', 'F#': 'F#4', 'G': 'G4', 'G#': 'G#4', 'A': 'A4', 'A#': 'A#4', 'B': 'B4', 'C2': 'C5'
            };

            const synthesizer = new Tone.Synth().toDestination();

            const noteLabelsDiv = document.getElementById('note-labels');
            const referenceNotesDiv = document.getElementById('reference-notes');
            const draggableEmojisDiv = document.getElementById('draggable-emojis');
            const feedbackMessage = document.getElementById('feedback-message');
            const resetButton = document.getElementById('reset-button');
            const confirmButton = document.getElementById('confirm-button');
            
            let selectedEmojiElement = null;
            let lastPlayedRefNote = null;

            // Function to generate the HTML for a single note
            function createNoteElement(noteName, emoji, color, isEmojiMovable = false) {
                const container = document.createElement('div');
                container.classList.add('note-container');

                const noteDiv = document.createElement('div');
                noteDiv.classList.add('note');
                noteDiv.dataset.note = noteName;

                if (isEmojiMovable) {
                    noteDiv.classList.add('note-emoji');
                    noteDiv.textContent = emoji;
                } else {
                    noteDiv.classList.add(`note-${color}`);
                    noteDiv.textContent = emoji;
                }

                container.appendChild(noteDiv);
                
                if (!isEmojiMovable) {
                    const label = document.createElement('div');
                    label.classList.add('note-label');
                    label.textContent = noteName;
                    container.appendChild(label);
                }

                return container;
            }

            // Shuffle an array
            function shuffleArray(array) {
                const shuffled = [...array];
                for (let i = shuffled.length - 1; i > 0; i--) {
                    const j = Math.floor(Math.random() * (i + 1));
                    [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
                }
                return shuffled;
            }

            // Function to initialize the game
            function initializeGame() {
                noteLabelsDiv.innerHTML = '';
                referenceNotesDiv.innerHTML = '';
                draggableEmojisDiv.innerHTML = '';
                feedbackMessage.textContent = '';
                feedbackMessage.classList.remove('bg-green-600', 'bg-red-600');
                selectedEmojiElement = null;
                lastPlayedRefNote = null;
                confirmButton.disabled = true;
                
                // Row 1: Note Labels
                noteOrder.forEach(noteName => {
                    const labelDiv = document.createElement('div');
                    labelDiv.classList.add('note-container');
                    labelDiv.innerHTML = `<div class="note-label">${noteName}</div>`;
                    noteLabelsDiv.appendChild(labelDiv);
                });

                // Row 2: Reference Notes with Emojis
                noteOrder.forEach((noteName, index) => {
                    const noteElement = createNoteElement(noteName, singleEmoji, noteColors[index]);
                    referenceNotesDiv.appendChild(noteElement);
                });

                // Add event listeners to play sound and set last played note
                referenceNotesDiv.querySelectorAll('.note').forEach(noteDiv => {
                    noteDiv.addEventListener('click', () => {
                        const noteName = noteDiv.dataset.note;
                        if (noteFrequencies[noteName]) {
                            synthesizer.triggerAttackRelease(noteFrequencies[noteName], '8n');
                            lastPlayedRefNote = noteName;
                            confirmButton.disabled = !(selectedEmojiElement && lastPlayedRefNote);
                        }
                    });
                });

                // Row 4: Movable Emojis
                const shuffledNotes = shuffleArray(noteOrder);
                shuffledNotes.forEach(noteName => {
                    const noteElement = createNoteElement(noteName, singleEmoji, 'gray', true);
                    draggableEmojisDiv.appendChild(noteElement);
                });
                
                // Add event listeners for tap-to-select
                addTapListeners();
            }

            function addTapListeners() {
                // Event listener for selecting an emoji from the 4th row
                draggableEmojisDiv.addEventListener('click', (event) => {
                    const clickedElement = event.target.closest('.note-emoji');
                    if (!clickedElement || clickedElement.classList.contains('correct-note')) {
                        return;
                    }

                    if (selectedEmojiElement) {
                        selectedEmojiElement.classList.remove('selected');
                    }
                    selectedEmojiElement = clickedElement;
                    selectedEmojiElement.classList.add('selected');

                    // Play the sound of the selected emoji
                    const noteName = clickedElement.dataset.note;
                    if (noteFrequencies[noteName]) {
                        synthesizer.triggerAttackRelease(noteFrequencies[noteName], '8n');
                    }
                    confirmButton.disabled = !(selectedEmojiElement && lastPlayedRefNote);
                });
            }

            confirmButton.addEventListener('click', () => {
                if (!selectedEmojiElement || !lastPlayedRefNote) {
                    return;
                }

                const selectedNote = selectedEmojiElement.dataset.note;
                
                if (selectedNote === lastPlayedRefNote) {
                    feedbackMessage.textContent = '';
                    feedbackMessage.classList.remove('bg-green-600', 'bg-red-600');
                    
                    selectedEmojiElement.classList.add('correct-note');
                    
                } else {
                    feedbackMessage.textContent = 'Try Again!';
                    feedbackMessage.classList.add('bg-red-600');
                    feedbackMessage.classList.remove('bg-green-600');
                    selectedEmojiElement.classList.add('incorrect-match');
                    setTimeout(() => selectedEmojiElement.classList.remove('incorrect-match'), 500);
                }
                
                selectedEmojiElement.classList.remove('selected');
                selectedEmojiElement = null;
                lastPlayedRefNote = null;
                confirmButton.disabled = true;
            });

            resetButton.addEventListener('click', initializeGame);

            // Initial game setup
            initializeGame();
        });
    </script>
</body>
</html>

